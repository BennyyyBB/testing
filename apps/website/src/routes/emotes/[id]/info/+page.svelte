<script lang="ts">
	import EmoteTabs from "$/components/layout/emote-tabs.svelte";
	import type { PageData } from "./$types";
	import { gqlClient } from "$/lib/gql";
	import { graphql } from "$/gql";
	import type { EmoteScores } from "$/gql/graphql";
	import { t } from "svelte-i18n";
	import { numberFormat, exactNumberFormat } from "$/lib/utils";
	import { Users, ChartLineUp, Trophy } from "phosphor-svelte";

	let { data }: { data: PageData } = $props();

	type InfoStats = {
		channelCount: number;
		scores: EmoteScores;
		trendingWeekRank: number | null;
		topAllTimeRank: number | null;
	};

	async function loadStats(id: string): Promise<InfoStats> {
		try {
			return await loadStatsUnsafe(id);
		} catch (error) {
			console.error("Failed to load emote info stats", error);
			throw error;
		}
	}

	async function loadStatsUnsafe(id: string): Promise<InfoStats> {
		const res = await gqlClient()
			.query(
				graphql(`
					query EmoteInfoStats($id: Id!) {
						emotes {
							emote(id: $id) {
								channels(page: 1, perPage: 1) {
									totalCount
								}
								scores {
									topAllTime
									topDaily
									topWeekly
									topMonthly
									trendingDay
									trendingWeek
									trendingMonth
								}
								trendingWeekRank: ranking(ranking: TRENDING_WEEKLY)
								topAllTimeRank: ranking(ranking: TOP_ALL_TIME)
							}
						}
					}
				`),
				{ id },
			)
			.toPromise();

		if (res.error || !res.data || !res.data.emotes.emote) {
			throw res.error;
		}

		return {
			channelCount: res.data.emotes.emote.channels.totalCount,
			scores: res.data.emotes.emote.scores,
			trendingWeekRank: res.data.emotes.emote.trendingWeekRank ?? null,
			topAllTimeRank: res.data.emotes.emote.topAllTimeRank ?? null,
		};
	}

	let stats = $derived(loadStats(data.id));
</script>

<div class="navigation">
	{#await data.streamed.emote then emote}
		<EmoteTabs id={emote.id} />
	{/await}
</div>

<div class="stats">
	{#await stats}
		<div class="stat loading-animation"></div>
		<div class="stat loading-animation"></div>
	{:then stats}
		<div class="stat" title={$t("pages.emote.info.channels_tooltip")}>
			<Users size="1.25rem" />
			<div>
				<span class="value">{exactNumberFormat().format(stats.channelCount)}</span>
				<span class="label">{$t("pages.emote.info.used_by_channels")}</span>
			</div>
		</div>
		{#if stats.trendingWeekRank !== null || stats.scores.trendingWeek > 0}
			<div class="stat" title={$t("pages.emote.info.trending_tooltip")}>
				<ChartLineUp size="1.25rem" />
				<div>
					<span class="value">
						{#if stats.trendingWeekRank !== null}
							#{numberFormat().format(stats.trendingWeekRank)}
						{:else}
							{$t("pages.emote.info.not_ranked")}
						{/if}
					</span>
					<span class="label">{$t("pages.emote.info.trending_this_week")}</span>
					<span class="raw-score">
						{$t("pages.emote.info.raw_score", {
							values: { score: numberFormat().format(stats.scores.trendingWeek) },
						})}
					</span>
				</div>
			</div>
		{/if}
		{#if stats.topAllTimeRank !== null || stats.scores.topAllTime > 0}
			<div class="stat" title={$t("pages.emote.info.top_all_time_tooltip")}>
				<Trophy size="1.25rem" />
				<div>
					<span class="value">
						{#if stats.topAllTimeRank !== null}
							#{numberFormat().format(stats.topAllTimeRank)}
						{:else}
							{$t("pages.emote.info.not_ranked")}
						{/if}
					</span>
					<span class="label">{$t("pages.emote.info.top_all_time")}</span>
					<span class="raw-score">
						{$t("pages.emote.info.raw_score", {
							values: { score: numberFormat().format(stats.scores.topAllTime) },
						})}
					</span>
				</div>
			</div>
		{/if}
	{:catch}
		<p class="stats-error">{$t("pages.emote.info.stats_error")}</p>
	{/await}
</div>

<p class="usage-note">{$t("pages.emote.info.usage_note")}</p>

<style lang="scss">
	.navigation {
		display: flex;
		align-items: center;
		justify-content: space-between;
		gap: 0.5rem;
	}

	.stats {
		margin-top: 1.5rem;

		display: flex;
		flex-wrap: wrap;
		gap: 0.75rem;

		.stat {
			flex: 1 1 10rem;
			min-height: 3.5rem;

			display: flex;
			align-items: center;
			gap: 0.75rem;

			padding: 0.75rem 1rem;
			background-color: var(--bg-light);
			border-radius: 0.5rem;
			color: var(--primary);
			cursor: help;

			div {
				display: flex;
				flex-direction: column;
			}

			.value {
				font-size: 1rem;
				font-weight: 700;
				color: var(--text);
			}

			.label {
				font-size: 0.75rem;
				color: var(--text-light);
			}

			.raw-score {
				margin-top: 0.2rem;
				font-size: 0.7rem;
				color: var(--text-light);
				opacity: 0.75;
			}
		}
	}

	.usage-note {
		margin-top: 0.75rem;
		font-size: 0.75rem;
		color: var(--text-light);
	}

	.stats-error {
		margin-top: 0.75rem;
		font-size: 0.75rem;
		color: var(--danger);
	}
</style>
