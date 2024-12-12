<script lang="ts">
	import "$lib/default.scss";
    import type { RedditCommentData } from "$lib/redditTypes";
    import DateDisplay from "./DateDisplay.svelte";

	export let data: RedditCommentData;
</script>

<div class="pane" class:removed={data?._meta?.removal_type}>
	<div class="header">
		<a href={`https://reddit.com/r/${data.subreddit}`} target="_blank">r/{data.subreddit} </a>
		<span>by </span>
		<a href={`https://reddit.com/u/${data.author}`} target="_blank">u/{data.author} </a>
		<span><DateDisplay date={new Date(data.created_utc * 1000)} /></span>
		<span> | </span>
		<span>{data.score} 🠉</span>
	</div>
	<div class="body long-text">
		{@html data.body_html}
	</div>
	<div class="reddit-link">
		<span>Source link:&nbsp;</span>
		<a href={`https://reddit.com${data.permalink}`} target="_blank" class="long-url">{`https://reddit.com${data.permalink}`}</a>
	</div>
	{#if data?._meta?.removal_type}
	<div class="removal-type">
		<span>Removed:&nbsp;{data._meta.removal_type}</span>
	</div>
	{/if}
</div>

<style lang="scss">
	.header {
		a {
			color: var(--primary);
		}
	}

	.reddit-link {
		display: flex;
		flex-direction: row;
		margin-top: 0.5rem;

		> * {
			white-space: nowrap;
			font-size: 0.8rem;
		}
	}

	.removal-type {
		margin-top: 0.5rem;

		> * {
			white-space: nowrap;
			font-size: 0.8rem;
		}
	}

	.removed {
		background: var(--user-admin-bg);
	}

	.body {
		margin-top: 0.5rem;
		margin-bottom: 0.5rem;
	}

	.long-url {
		overflow: hidden;
		text-overflow: ellipsis;
		white-space: nowrap;
		width: 100%;
		display: inline-block;
	}

	.long-text {
		overflow: hidden;
		text-overflow: ellipsis;
	}
</style>
