<!--
SPDX-FileCopyrightText: mk-go project
SPDX-License-Identifier: AGPL-3.0-only
-->

<!--
	mk-go: IP lookup audit history (#3106 / #3276).
	This component is mounted only from the authorized moderation-log tab. The
	server remains the source of truth for moderator, policy, and API-scope checks.
-->
<template>
<div class="_gaps_m">
	<MkInfo>{{ i18n.ts._mkgoIpLookupLog.about }}</MkInfo>

	<div :class="$style.status" aria-live="polite">{{ status }}</div>
	<MkInfo v-if="error && !errorWhilePaging" warn>{{ error }}</MkInfo>
	<div v-if="loading" :class="$style.placeholder"><MkLoading/></div>

	<template v-else-if="result">
		<MkInfo>{{ i18n.tsx._mkgoIpLookupLog.retentionNote({ n: result.retentionDays }) }}</MkInfo>
		<MkInfo v-if="entries.length === 0">{{ i18n.tsx._mkgoIpLookupLog.empty({ n: result.retentionDays }) }}</MkInfo>

		<div v-else class="_gaps_s">
			<MkTl :events="timeline" groupBy="d">
				<template #left="{ event }">
					<MkAvatar v-if="event.user" :user="event.user" style="width: 26px; height: 26px;"/>
					<i v-else class="ti ti-user-off"></i>
				</template>
				<template #right="{ event: entry }">
					<MkFolder :key="entry.id">
						<template #label>{{ kindLabel(entry.kind) }}</template>
						<template #caption><MkTime :time="entry.createdAt" mode="detail"/></template>

						<MkA v-if="entry.user" :to="`/admin/user/${entry.user.id}`" :class="$style.card">
							<MkUserCardMini :user="entry.user" :withChart="false"/>
						</MkA>
						<div v-else :class="$style.goneUser">
							{{ i18n.ts._mkgoIpLookupLog.userGone }}
							<span class="_monospace">{{ entry.userId }}</span>
						</div>

						<div :class="$style.facts">
							<MkKeyValue oneline>
								<template #key>{{ i18n.ts._mkgoIpLookupLog.at }}</template>
								<template #value><MkTime :time="entry.createdAt" mode="detail"/></template>
							</MkKeyValue>
							<MkKeyValue oneline>
								<template #key>{{ i18n.ts._mkgoIpLookupLog.kind }}</template>
								<template #value>{{ kindLabel(entry.kind) }}</template>
							</MkKeyValue>
							<MkKeyValue oneline>
								<template #key>{{ i18n.ts._mkgoIpLookupLog.subject }}</template>
								<template #value>
									<span v-if="entry.kind === 'ip'" class="_monospace">{{ entry.ip }}</span>
									<MkA v-else-if="entry.targetUser" :to="`/admin/user/${entry.targetUser.id}`">@{{ entry.targetUser.username }}</MkA>
									<span v-else class="_monospace">{{ entry.targetUserId }}</span>
								</template>
							</MkKeyValue>
							<MkKeyValue oneline>
								<template #key>{{ i18n.ts._mkgoIpLookupLog.period }}</template>
								<template #value>
									<template v-if="entry.sinceDays > 0">{{ i18n.tsx._mkgoIpLookupLog.periodDays({ n: entry.sinceDays }) }}</template>
									<template v-else>{{ i18n.ts._mkgoIpLookupLog.noPeriod }}</template>
								</template>
							</MkKeyValue>
							<MkKeyValue oneline>
								<template #key>{{ i18n.ts._mkgoIpLookupLog.resultCount }}</template>
								<template #value>{{ i18n.tsx._mkgoIpLookupLog.resultCountValue({ n: number(entry.resultCount) }) }}</template>
							</MkKeyValue>
						</div>
					</MkFolder>
				</template>
			</MkTl>
			<div :class="$style.caption">{{ i18n.ts._mkgoIpLookupLog.resultsNotRecorded }}</div>
		</div>

		<MkInfo v-if="error && errorWhilePaging" warn>{{ error }}</MkInfo>
		<MkButton v-if="result.hasMore" :disabled="loadingMore" @click="loadMore()">{{ i18n.ts.loadMore }}</MkButton>
	</template>
</div>
</template>

<script lang="ts" setup>
import { computed, onMounted, ref } from 'vue';
import type * as Misskey from 'misskey-js';
import MkButton from '@/components/MkButton.vue';
import MkFolder from '@/components/MkFolder.vue';
import MkInfo from '@/components/MkInfo.vue';
import MkKeyValue from '@/components/MkKeyValue.vue';
import MkTl from '@/components/MkTl.vue';
import MkUserCardMini from '@/components/MkUserCardMini.vue';
import { i18n } from '@/i18n.js';
import number from '@/filters/number.js';
import { misskeyApi } from '@/utility/misskey-api.js';
import { ipSearchErrorKind } from '@/utility/ip-search-result.js';

type IPLookupLogEntry = {
	id: string;
	user: Misskey.entities.UserLite | null;
	userId: string;
	kind: string;
	ip: string;
	targetUser: Misskey.entities.UserLite | null;
	targetUserId: string;
	sinceDays: number;
	resultCount: number;
	createdAt: string;
};

type IPLookupLogResponse = {
	retentionDays: number;
	limit: number;
	offset: number;
	hasMore: boolean;
	entries: IPLookupLogEntry[];
};

function api<T>(endpoint: string, params: Record<string, unknown> = {}): Promise<T> {
	return misskeyApi(endpoint as never, params as never) as unknown as Promise<T>;
}

const loading = ref(true);
const loadingMore = ref(false);
const error = ref<string | null>(null);
const errorWhilePaging = ref(false);
const result = ref<IPLookupLogResponse | null>(null);
const entries = ref<IPLookupLogEntry[]>([]);
const timeline = computed(() => entries.value.map(entry => ({
	id: entry.id,
	timestamp: new Date(entry.createdAt).getTime(),
	data: entry,
})));

let generation = 0;

async function load(offset: number) {
	const first = offset === 0;
	error.value = null;
	errorWhilePaging.value = false;
	const gen = ++generation;
	if (first) {
		loading.value = true;
		entries.value = [];
	} else {
		loadingMore.value = true;
	}
	try {
		const response = await api<IPLookupLogResponse>('admin/ip/lookup-log', { offset });
		if (gen !== generation) return;
		result.value = response;
		entries.value = first ? response.entries : mergeEntries(entries.value, response.entries);
	} catch (reason) {
		if (gen !== generation) return;
		if (first) {
			result.value = null;
			entries.value = [];
		}
		error.value = errorMessage(reason);
		errorWhilePaging.value = !first;
	} finally {
		if (gen === generation) {
			loading.value = false;
			loadingMore.value = false;
		}
	}
}

function errorMessage(reason: unknown): string {
	const kind = ipSearchErrorKind(reason, false);
	if (kind === 'pagingLimit') return i18n.ts._mkgoIpLookupLog.pagingLimit;
	return i18n.ts._mkgoIpSearch[kind];
}

function mergeEntries(current: IPLookupLogEntry[], incoming: IPLookupLogEntry[]): IPLookupLogEntry[] {
	const seen = new Set(current.map(entry => entry.id));
	return [...current, ...incoming.filter(entry => !seen.has(entry.id))];
}

function kindLabel(kind: string): string {
	switch (kind) {
		case 'ip': return i18n.ts._mkgoIpLookupLog.kindIp;
		case 'relatedAccounts': return i18n.ts._mkgoIpLookupLog.kindRelated;
		case 'userIps': return i18n.ts._mkgoIpLookupLog.kindUserIps;
		case 'signins': return i18n.ts._mkgoIpLookupLog.kindSignins;
		default: return kind;
	}
}

const status = computed(() => {
	if (loading.value) return i18n.ts._mkgoIpLookupLog.loading;
	if (error.value != null) return error.value;
	if (result.value == null) return '';
	if (entries.value.length === 0) return i18n.tsx._mkgoIpLookupLog.empty({ n: result.value.retentionDays });
	return i18n.tsx._mkgoIpLookupLog.found({ n: number(entries.value.length) });
});

function loadMore() {
	if (result.value == null) return Promise.resolve();
	return load(result.value.offset + result.value.limit);
}

onMounted(() => load(0));
</script>

<style lang="scss" module>
.placeholder {
	padding: 32px;
	text-align: center;
}

.status {
	position: absolute;
	width: 1px;
	height: 1px;
	margin: -1px;
	padding: 0;
	overflow: hidden;
	clip-path: inset(50%);
	white-space: nowrap;
}

.card {
	display: block;
}

.goneUser {
	font-size: 0.9em;
	opacity: 0.7;
}

.facts {
	margin-top: 8px;
	display: grid;
	gap: 4px;
}

.caption {
	font-size: 0.85em;
	opacity: 0.7;
}
</style>
