<script lang="ts">
	import { getContext, onMount, onDestroy } from 'svelte';

	import { tools } from '$lib/stores';
	import { getTools, getMCPServerResources, completeMCPResourceArgument } from '$lib/apis/tools';

	import Tooltip from '$lib/components/common/Tooltip.svelte';
	import Database from '$lib/components/icons/Database.svelte';

	const i18n = getContext('i18n');

	export let query = '';
	export let onSelect = (e) => {};

	let selectedIdx = 0;
	export let filteredItems = [];

	let items = [];

	// Template argument-filling mode: set when the user picks a resource
	// template (e.g. vik://projects/{project_id}); candidates come from the
	// server via MCP completion/complete.
	let active = null; // { server_id, server_name, name, uri, args, argIdx, baseQueryLen }
	let candidates = [];
	// Bumped per request so a slow completion resolving after a newer one
	// cannot overwrite fresh candidates.
	let completionSeq = 0;
	let completionDebounceTimer: ReturnType<typeof setTimeout>;

	// Type-driven completion: as the user types a path (e.g. vik://projects/Inbox/),
	// match it against templates and complete the current segment. Decoupled from
	// the click-driven `active` flow with its own seq/timer.
	let typedCandidates = [];
	let typedSeq = 0;
	let typedDebounceTimer: ReturnType<typeof setTimeout>;

	const parseTemplateArgs = (uriTemplate: string): string[] => {
		return [...uriTemplate.matchAll(/{([^}]+)}/g)].map((m) => m[1]);
	};

	// Split a template into literal and placeholder parts, in order.
	const templateParts = (uri: string) => {
		const parts = [];
		let last = 0;
		const re = /{([^}]+)}/g;
		let m;
		while ((m = re.exec(uri))) {
			if (m.index > last) parts.push({ lit: uri.slice(last, m.index) });
			parts.push({ ph: m[1] });
			last = m.index + m[0].length;
		}
		if (last < uri.length) parts.push({ lit: uri.slice(last) });
		return parts;
	};

	// Align typed text with a template's structure. Returns the resolved earlier
	// segments, the segment currently being typed, and its partial value — or
	// null if the typed text doesn't match this template's literal layout.
	const matchTyped = (typed: string, uri: string) => {
		const parts = templateParts(uri);
		let pos = 0;
		const resolved = {};
		for (let i = 0; i < parts.length; i++) {
			const p = parts[i];
			if (p.lit !== undefined) {
				const rest = typed.slice(pos);
				if (rest.startsWith(p.lit)) pos += p.lit.length;
				else if (p.lit.startsWith(rest)) return null; // still typing the literal
				else return null; // mismatch
			} else {
				const nextLit = parts[i + 1]?.lit;
				const rest = typed.slice(pos);
				if (nextLit === undefined) return { resolved, argName: p.ph, argValue: rest };
				const idx = rest.indexOf(nextLit);
				if (idx === -1) return { resolved, argName: p.ph, argValue: rest };
				resolved[p.ph] = rest.slice(0, idx);
				pos += idx;
			}
		}
		return { resolved, argName: null, argValue: '' };
	};

	// Best template for the typed text: most resolved segments wins, then having
	// a segment in progress. Lets `vik://projects/Inbox/` prefer the deeper
	// {project}/{task} template over the shallow {project} one.
	const matchBest = (typed: string) => {
		let best = null;
		for (const it of items) {
			if (!it.template) continue;
			const m = matchTyped(typed, it.uri);
			if (!m) continue;
			const score = Object.keys(m.resolved).length * 2 + (m.argName ? 1 : 0);
			if (!best || score > best._score) best = { ...m, template: it, _score: score };
		}
		return best;
	};

	const fetchTyped = async () => {
		const tm = typedMatch;
		if (!tm || !tm.argName) {
			typedCandidates = [];
			return;
		}
		const seq = ++typedSeq;
		const res = await completeMCPResourceArgument(
			localStorage.token,
			tm.template.server_id,
			tm.template.uri,
			tm.argName,
			tm.argValue,
			tm.resolved
		).catch(() => null);
		if (seq !== typedSeq) return;
		typedCandidates = res?.values ?? [];
	};

	const scheduleTyped = () => {
		clearTimeout(typedDebounceTimer);
		typedDebounceTimer = setTimeout(() => {
			fetchTyped();
		}, 200);
	};

	// Recompute the typed match whenever the query changes (browse mode only).
	$: typedMatch = !active && query ? matchBest(query) : null;

	// Fetch completions for the typed segment. Plain function call keeps the
	// timer/candidates out of this block's dependency graph (no reactive loop).
	$: {
		typedMatch;
		scheduleTyped();
	}

	const loadResources = async () => {
		if ($tools === null) {
			tools.set(await getTools(localStorage.token).catch(() => []));
		}

		const mcpServers = ($tools ?? []).filter((tool) =>
			String(tool.id).startsWith('server:mcp:')
		);

		const results = await Promise.all(
			mcpServers.map(async (server) => {
				const serverId = String(server.id).slice('server:mcp:'.length);
				const res = await getMCPServerResources(localStorage.token, serverId).catch(() => null);

				return [
					...(res?.resources ?? []).map((resource) => ({
						template: false,
						server_id: serverId,
						server_name: server.name,
						uri: resource.uri,
						name: resource.name || resource.uri,
						description: resource.description || ''
					})),
					...(res?.templates ?? []).map((template) => ({
						template: true,
						server_id: serverId,
						server_name: server.name,
						uri: template.uriTemplate,
						name: template.name || template.uriTemplate,
						description: template.description || ''
					}))
				];
			})
		);

		items = results.flat();
	};

	onMount(loadResources);

	onDestroy(() => {
		clearTimeout(completionDebounceTimer);
		clearTimeout(typedDebounceTimer);
	});

	const getCompletions = async () => {
		if (!active) {
			return;
		}

		const seq = ++completionSeq;
		const argValue = query.slice(active.baseQueryLen);
		const argName = active.args[active.argIdx];

		const res = await completeMCPResourceArgument(
			localStorage.token,
			active.server_id,
			active.template,
			argName,
			argValue,
			active.resolved ?? {}
		).catch(() => null);

		// Ignore results from a superseded request
		if (seq !== completionSeq || !active) {
			return;
		}

		candidates = res?.values ?? [];
	};

	// Debounce wrapper. The timer lives inside this function (not a reactive
	// block) so reading/writing it never creates a reactive dependency loop.
	const scheduleCompletions = () => {
		clearTimeout(completionDebounceTimer);
		completionDebounceTimer = setTimeout(() => {
			getCompletions();
		}, 200);
	};

	// Re-run when the typed argument text changes. Referencing active/query in
	// the block tracks them as deps; the function call keeps the timer out of
	// the dependency graph.
	const onArgInput = () => {
		if (!active) {
			return;
		}
		if (query.length < active.baseQueryLen) {
			// User erased past the selection point — back to browsing
			completionSeq++; // cancel any in-flight completion
			clearTimeout(completionDebounceTimer);
			active = null;
			candidates = [];
		} else {
			scheduleCompletions();
		}
	};

	$: {
		active;
		query;
		onArgInput();
	}

	$: argValue = active ? query.slice(active.baseQueryLen) : '';

	$: filteredItems = active
		? [
				...candidates
					.filter((value) => argValue === '' || String(value).includes(argValue))
					.map((value) => ({
						candidate: true,
						value: String(value),
						uri: active.uri.replace(`{${active.args[active.argIdx]}}`, String(value))
					})),
				...(argValue !== '' && !candidates.map(String).includes(argValue)
					? [
							{
								candidate: true,
								typed: true,
								value: argValue,
								uri: active.uri.replace(`{${active.args[active.argIdx]}}`, argValue)
							}
						]
					: []),
				// "Use as-is" stays last so Enter defaults to the first candidate
				...(active.finalizeUri ? [{ finalize: true, uri: active.finalizeUri }] : [])
			]
		: typedMatch && typedMatch.argName
			? // Type-driven: completing the segment the user is typing
				[
					...typedCandidates
						.filter(
							(v) =>
								typedMatch.argValue === '' ||
								String(v).toLowerCase().includes(typedMatch.argValue.toLowerCase())
						)
						.map((v) => ({
							typedCandidate: true,
							match: typedMatch,
							value: String(v),
							uri: substituteResolved(typedMatch.template.uri, {
								...typedMatch.resolved,
								[typedMatch.argName]: String(v)
							})
						})),
					...(typedMatch.argValue !== '' && !typedCandidates.map(String).includes(typedMatch.argValue)
						? [
								{
									typedCandidate: true,
									match: typedMatch,
									typed: true,
									value: typedMatch.argValue,
									uri: substituteResolved(typedMatch.template.uri, {
										...typedMatch.resolved,
										[typedMatch.argName]: typedMatch.argValue
									})
								}
							]
						: [])
				]
			: items.filter((item) => {
					const q = (query ?? '').toLowerCase();
					return (
						q === '' ||
						item.name.toLowerCase().includes(q) ||
						item.uri.toLowerCase().includes(q) ||
						item.server_id.toLowerCase().includes(q)
					);
				});

	$: if (query) {
		selectedIdx = 0;
	}

	const substituteResolved = (templateUri, resolved) => {
		let u = templateUri;
		for (const [k, v] of Object.entries(resolved)) {
			u = u.replace(`{${k}}`, String(v));
		}
		return u;
	};

	// Find a template on the same server that extends the current one with more
	// path segments (e.g. vik://projects/{project}/{task} extends
	// vik://projects/{project}), so path drilling continues automatically.
	const findDeeperTemplate = (templateUri, serverId, resolved) => {
		const deeper = items
			.filter(
				(it) =>
					it.template &&
					it.server_id === serverId &&
					it.uri !== templateUri &&
					it.uri.startsWith(templateUri + '/')
			)
			.sort((a, b) => parseTemplateArgs(a.uri).length - parseTemplateArgs(b.uri).length);
		return deeper[0] ?? null;
	};

	const finalize = (uri) => {
		const data = {
			server_id: active.server_id,
			server_name: active.server_name,
			uri,
			name: uri
		};
		active = null;
		candidates = [];
		onSelect({ type: 'resource', data });
	};

	// Fill the current argument of `active` with `value`, then advance to the
	// next argument, drill into a deeper template, or finalize. Shared by the
	// click-driven candidates and the type-driven candidates.
	const selectCandidateValue = (value) => {
		const argName = active.args[active.argIdx];
		const nextUri = active.uri.replace(`{${argName}}`, String(value));
		const nextResolved = { ...active.resolved, [argName]: value };

		if (active.argIdx + 1 < active.args.length) {
			// More arguments in this template — advance to the next one
			active = {
				...active,
				uri: nextUri,
				resolved: nextResolved,
				argIdx: active.argIdx + 1,
				baseQueryLen: query.length
			};
			candidates = [];
			getCompletions();
			return;
		}

		// Last argument filled. If a deeper template extends this path, continue
		// into it (carrying resolved args as completion context); else finalize.
		const deeper = findDeeperTemplate(active.template, active.server_id, nextResolved);
		if (deeper) {
			const args = parseTemplateArgs(deeper.uri);
			const nextIdx = args.findIndex((a) => !(a in nextResolved));
			active = {
				server_id: active.server_id,
				server_name: active.server_name,
				name: deeper.name,
				template: deeper.uri,
				uri: substituteResolved(deeper.uri, nextResolved),
				args,
				argIdx: nextIdx < 0 ? args.length - 1 : nextIdx,
				resolved: nextResolved,
				finalizeUri: nextUri, // let the user stop at this shallower resource
				baseQueryLen: query.length
			};
			candidates = [];
			getCompletions();
			return;
		}

		finalize(nextUri);
	};

	const selectItem = (item) => {
		if (item.finalize) {
			finalize(item.uri);
			return;
		}
		if (item.typedCandidate) {
			// User typed a path; seed `active` from the matched template position
			// then reuse the candidate-selection flow.
			const tm = item.match;
			const args = parseTemplateArgs(tm.template.uri);
			active = {
				server_id: tm.template.server_id,
				server_name: tm.template.server_name,
				name: tm.template.name,
				template: tm.template.uri,
				uri: substituteResolved(tm.template.uri, tm.resolved),
				args,
				argIdx: args.indexOf(tm.argName),
				resolved: tm.resolved,
				baseQueryLen: query.length
			};
			selectCandidateValue(item.value);
			return;
		}
		if (item.candidate) {
			selectCandidateValue(item.value);
		} else if (item.template) {
			active = {
				server_id: item.server_id,
				server_name: item.server_name,
				name: item.name,
				template: item.uri, // original uriTemplate — the completion ref
				uri: item.uri, // progressively substituted for display/result
				args: parseTemplateArgs(item.uri),
				argIdx: 0,
				resolved: {}, // arg name -> chosen value, sent as completion context
				baseQueryLen: query.length
			};
			candidates = [];
			getCompletions();
		} else {
			onSelect({ type: 'resource', data: item });
		}
	};

	export const selectUp = () => {
		selectedIdx = Math.max(0, selectedIdx - 1);
	};

	export const selectDown = () => {
		selectedIdx = Math.min(selectedIdx + 1, filteredItems.length - 1);
	};

	export const select = async () => {
		const item = filteredItems[selectedIdx];
		if (item) {
			selectItem(item);
		}
	};
</script>

<div class="px-2 text-xs text-gray-500 py-1">
	{#if active}
		{active.server_name} — {active.args[active.argIdx]}
	{:else}
		{$i18n.t('MCP Resources')}
	{/if}
</div>

{#if filteredItems.length > 0}
	{#each filteredItems as item, itemIdx}
		<Tooltip
			content={item.finalize
				? $i18n.t('Use this resource as-is')
				: item.candidate || item.typedCandidate
					? item.typed
						? $i18n.t('Use typed value')
						: item.uri
					: item.description || item.uri}
			placement="top-start"
		>
			<button
				class="px-2.5 py-1.5 rounded-xl w-full text-left {itemIdx === selectedIdx
					? 'bg-gray-50 dark:bg-gray-800 selected-command-option-button'
					: ''}"
				type="button"
				on:click={() => {
					selectItem(item);
				}}
				on:mousemove={() => {
					selectedIdx = itemIdx;
				}}
				on:focus={() => {}}
				data-selected={itemIdx === selectedIdx}
			>
				<div class="flex text-black dark:text-gray-100 line-clamp-1 items-center">
					<div class="flex items-center justify-center size-5 mr-2 shrink-0">
						<Database className="size-4" />
					</div>
					{#if item.finalize}
						<div class="truncate">
							{item.uri}
						</div>
						<div class="ml-2 text-xs text-gray-500 truncate shrink-0">
							{$i18n.t('use as-is')}
						</div>
					{:else if item.candidate || item.typedCandidate}
						<div class="truncate">
							{item.uri}
						</div>
						{#if item.typed}
							<div class="ml-2 text-xs text-gray-500 truncate shrink-0">
								{$i18n.t('typed')}
							</div>
						{/if}
					{:else}
						<div class="truncate">
							{item.uri}
						</div>
						<div class="ml-2 text-xs text-gray-500 truncate shrink-0">
							{item.template ? `${item.name} · ${item.server_name}` : item.server_name}
						</div>
					{/if}
				</div>
			</button>
		</Tooltip>
	{/each}
{/if}
