<script lang="ts">
	import { onMount } from "svelte";
	import "$lib/default.scss";
	import TextField from "$lib/components/TextField.svelte";
    import OptionSelector from "$lib/components/OptionSelector.svelte";
    import type { RedditCommentData, RedditPostData } from "$lib/redditTypes";
    import RedditPost from "$lib/components/RedditPost.svelte";
    import RedditComment from "$lib/components/RedditComment.svelte";
	import homeSvg from "$lib/images/home.svg";
	import settingsSvg from "$lib/images/settings.svg";
	import Preferences from "./Preferences.svelte"

	enum Function {
		PostIds="post_ids",
		CommentIds="comment_ids",
		PostsSearch="posts_search",
		CommentsSearch="comments_search",
		ThreadSearch="thread_search",
		RepliesSearch="replies_search",
	}

	const appId = "N3bHM42Rmlc3sQ";
	const authHeader = `Basic ${btoa(appId + ":")}`;
	const clientId = "syqnq8auSEvxwe2Crb1u";
	let accessToken = null;
	let newlyRemoved = {};
	let cachedOPs = {};

	async function fetchRedditAuth(bodyParams, additionalHeaders = {}) {
		try {
			const response = await fetch("https://www.reddit.com/api/v1/access_token", {
				method: "POST",
				headers: {
					Authorization: authHeader,
					"Content-Type": "application/x-www-form-urlencoded",
					...additionalHeaders,
				},
				body: new URLSearchParams(bodyParams),
			});

			return await response.json();
		} catch (e) {
			console.error("Reddit API error", e);
			return { error: "Error getting access token" };
		}
	}

	async function getAccessToken() {
		const response = await fetchRedditAuth({
			grant_type: "https://oauth.reddit.com/grants/installed_client",
			device_id: clientId,
		});
		accessToken = response?.access_token;
		return response;
	}

	const randomStringAlphabet = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789";
	function randomString(length: number): string {
		let randStr = "";
		for (let i = 0; i < length; ++i)
			randStr += randomStringAlphabet[Math.floor(Math.random() * randomStringAlphabet.length)];
		return randStr;
	}

	async function oauth2Request(path, id, forced = false, options = {}) {
		if (!accessToken) return;
		if (id in newlyRemoved && !forced) return;
		if (forced && id in cachedOPs) {
			posts = [cachedOPs[id]];
			return;
		}
		const parameters = new URLSearchParams();
		parameters.append("sr_detail", "true")
		parameters.append("raw_json", "1");
		parameters.append("rand", randomString(10));
		let fetchOptions = {
			...options,
			headers: new Headers({
				Authorization: `Bearer ${accessToken}`,
			}),
		};
		const response = await fetch(`https://oauth.reddit.com${path}?${parameters.toString()}`, fetchOptions);
		const json = await response.json();
		const post = json[0].data.children[0].data;
		newlyRemoved[id] = post.removed_by_category || post._meta?.removal_type || (post.author == "[deleted]" && post.author);
		if (forced) {
			cachedOPs[id] = post;
			posts = [post];
		}
	}

	getAccessToken();

	let fun = Function.PostsSearch;
	// search parameters
	let subreddit = "";
	let author = "";
	let after = "";
	let before = "";
	let limit = "10";
	let sort: "asc"|"desc" = "desc";
	// post search parameters
	let over18: boolean|null = null;
	let spoiler: boolean|null = null;
	let title = ""
	let selftext = ""
	let url = "";
	// comment search parameters
	let linkId = "";
	let parentId = "";
	let body = "";

	let showSettings = false;

	let loading = false;
	let error: string|null = null;
	let posts: RedditPostData[]|null = null;
	let comments: RedditCommentData[]|null = null;
	let replies: RedditCommentRepliesData = {};
	$: currentData = fun === Function.PostsSearch ? posts : null ?? fun === Function.CommentsSearch ? comments : null ?? fun === Function.ThreadSearch ? comments : null;
	let previousHistory: string[] = [];

	onMount(() => {
		const params = new URLSearchParams(location.search);
		const funStr = params.get("fun");
		if (Object.values(Function).includes(funStr as any))
			fun = funStr as Function;
		subreddit = params.get("subreddit") || "";
		author = params.get("author") || "";
		after = params.get("after") || "";
		before = params.get("before") || "";
		limit = params.get("limit") || "10";
		sort = params.get("sort") as "asc"|"desc" || "desc";
		over18 = params.get("over_18") == "true" ? true : params.get("over_18") == "false" ? false : null;
		spoiler = params.get("spoiler") == "true" ? true : params.get("spoiler") == "false" ? false : null;
		title = params.get("title") || "";
		selftext = params.get("selftext") || "";
		url = params.get("url") || "";
		linkId = params.get("link_id") || "";
		parentId = params.get("parent_id") || "";
		body = params.get("body") || "";
	});

	function verifyLimit(limitStr: string): string|null {
		if (limitStr.length == 0)
			return null;
		const limit = Number(limitStr);
		if (isNaN(limit))
			return "Invalid number";
		if (limit < 1 || limit > 100)
			return "Must be between 1 and 100";
		return null;
	}

	async function search(_: any, clearPrevious = true, activeFun: any = null) {
		if (clearPrevious)
			previousHistory = [];
		activeFun = activeFun ?? fun;
		error = null;
		loading = true;
		const params = new URLSearchParams();
		let paramNames: [string, string][] = [];
		let endpoint: string;
		if (activeFun == Function.PostsSearch) {
			endpoint = "posts";
			paramNames = [
				["subreddit", subreddit],
				["author", author],
				["after", after],
				["before", before],
				["limit", limit],
				["sort", sort],
				["over_18", over18 != null ? over18.toString() : ""],
				["spoiler", spoiler != null ? spoiler.toString() : ""],
				["title", title],
				["selftext", selftext],
				["url", url],
			];
		}
		else if (activeFun == Function.CommentsSearch) {
			endpoint = "comments";
			paramNames = [
				["subreddit", subreddit],
				["author", author],
				["after", after],
				["before", before],
				["limit", _?.limit ?? limit],
				["sort", sort],
				["link_id", linkId],
				["parent_id", parentId],
				["body", body],
			];
		}
		else if (activeFun == Function.ThreadSearch) {
			endpoint = "posts";
			paramNames = [
				["subreddit", subreddit],
				["author", author],
				["after", ""],
				["before", ""],
				["limit", "5"],
				["sort", sort],
				["over_18", over18 != null ? over18.toString() : ""],
				["spoiler", spoiler != null ? spoiler.toString() : ""],
				["title", title],
				["selftext", selftext],
				["url", url],
			];
		}
		else if (activeFun == Function.RepliesSearch) {
			endpoint = "comments";
			paramNames = [
				["subreddit", subreddit],
				["author", author],
				["after", ""],
				["before", ""],
				["limit", _?.limit ?? limit], // TODO: if we got exactly 10 replies, add a "load more" button
				["sort", sort],
				["link_id", _?.parentLinkId],
				["parent_id", _?.parentCommentId],
				["body", body],
			];
		}
		else {
			error = "Not implemented";
			loading = false;
			return;
		}
		const ownUrlParams = new URLSearchParams();
		ownUrlParams.append("fun", fun);
		for (const [name, value] of paramNames) {
			if (activeFun == Function.ThreadSearch && name == "limit") {
				params.append(name, value);
				ownUrlParams.append(name, limit);
			}
			else if (name == "parent_id" && value == "0") {
				params.append(name, "");
				ownUrlParams.append(name, value);
			}
			else if (value && value.length > 0) {
				params.append(name, value);
				ownUrlParams.append(name, value);
			}
		}
		const newOwnUrl = new URL(location.href);
		newOwnUrl.search = ownUrlParams.toString();
		if (fun != Function.ThreadSearch || activeFun == Function.ThreadSearch) {
			history.replaceState(null, "", newOwnUrl.toString());
		}
		params.append("md2html", "true");
		params.append("meta-app", "search-tool");
		
		const requestUrl = `https://arctic-shift.photon-reddit.com/api/${endpoint}/search?${params.toString()}`;
		try {
			const response = await fetch(requestUrl);
			let data;
			try {
				data = await response.json();
			} catch (e) {
				if (!response.ok) {
					error = `Error ${response.status} ${response.statusText}`;
					loading = false;
				}
				else {
					error = (e as Error).message;
					loading = false;
				}
				return;
			}
			if (data.error) {
				error = data.error;
				loading = false;
				return;
			}
			if (!data.data) {
				error = "No data";
				loading = false;
				return;
			}
			if (activeFun == Function.PostsSearch || activeFun == Function.ThreadSearch) {
				posts = data.data;
				if (fun == Function.ThreadSearch) {
					linkId = url.replace(/.*?\/comments\//, "").replace(/\/.*/, "")
					for (const post of posts) {
						if (post.id == linkId) {
							posts = [post];
						}
					}
					if (posts !== null && (posts.length == 0 || posts[0].id !== linkId)) oauth2Request(url.replace(/.*?\/r\//, "/r/"), linkId, true)
					parentId = "0"
					// TODO: don't override global values here ^
					search({parentLinkId: linkId, parentCommentId: ""}, false, Function.CommentsSearch)
				}
				for (const post of posts) {
					if (!(post.removed_by_category || post._meta?.removal_type)) {
						oauth2Request(post.permalink, post.id);
					}
				}
			}
			else if (activeFun == Function.CommentsSearch) {
				comments = data.data;
				if (fun == Function.ThreadSearch && comments) {
					for (const comment of comments) {
						if (!(comment.id in replies)) {
							// TODO: proper dedupe for when we allow loading extra replies
							search({parentLinkId: _?.parentLinkId, parentCommentId: comment.id}, false, Function.RepliesSearch)
						}
					}
				}
			}
			else if (activeFun == Function.RepliesSearch) {
				const replyData = data.data;
				if (replyData) {
					replies[_?.parentCommentId] = [];
					for (const reply of replyData) {
						replies[_?.parentCommentId].push(reply);
						search({parentLinkId: _?.parentLinkId, parentCommentId: reply.id}, false, Function.RepliesSearch)
					}
				}
			}

		}
		catch (e) {
			error = (e as Error).message;
		}
		loading = false;
	}

	function download() {
		let data: object[]|null = null;
		let name: string|null = null;
		if (fun == Function.PostsSearch) {
			data = posts;
			name = "posts";
		}
		else if (fun == Function.CommentsSearch) {
			data = comments;
			name = "comments";
		}
		else if (fun == Function.ThreadSearch) {
			data = [posts, comments, replies];
			name = "thread";
		}
		if (!data || !name)
			return;
		const blob = new Blob([JSON.stringify(data, null, "\t")], { type: "application/json" });
		const url = URL.createObjectURL(blob);
		const a = document.createElement("a");
		a.href = url;
		a.download = `${name}.json`;
		a.click();
	}

	function searchPagination(changeParameters: (data: RedditPostData[]|RedditCommentData[]) => void) {
		if (!currentData)
			return;
		changeParameters(currentData);
		search(null, false);
	}

	function searchNext() {
		searchPagination((data) => {
			const lastTimestamp = data[data.length - 1].created_utc;
			let firstTimestamp = data[0].created_utc;
			const lastDate = new Date(lastTimestamp * 1000).toISOString().replace(/\.\d+Z$/, "");
			if (sort === "desc") {
				before = lastDate;
				after = "";
				firstTimestamp += 1;
			}
			else {
				after = lastDate;
				before = "";
				firstTimestamp -= 1;
			}
			const firstDate = new Date(firstTimestamp * 1000).toISOString().replace(/\.\d+Z$/, "");
			previousHistory = [...previousHistory, firstDate];
		});
	}

	function searchPrevious() {
		searchPagination((data) => {
			const timestamp = previousHistory[previousHistory.length - 1];
			if (!timestamp)
				return;
			previousHistory = previousHistory.slice(0, -1);
			if (sort === "desc") {
				before = timestamp;
				after = "";
			}
			else {
				after = timestamp;
				before = "";
			}
		});
	}
</script>

<svelte:head>
	<title>Search through reddit data</title>
	<meta name="description" content="Search through reddit post and comment, using parameters like subreddit, author, date, body, etc.">
</svelte:head>

<div class="search">
	<div class="parameters pane">
		<div class="row title-row">
			<h1>Search</h1>
			<a href="/" class="home-button">
				<img src={homeSvg} alt="home" />
			</a>
		</div>
		<p>
			Search for posts or comments. All filter parameters are optional.
		</p>
		<div class="gap"></div>
		<OptionSelector
			options={[
				{ value: Function.PostsSearch, label: "Posts Search" },
				{ value: Function.CommentsSearch, label: "Comments Search" },
				{ value: Function.ThreadSearch, label: "Thread Search" },
				// { value: Function.PostIds, label: "Post IDs" },
				// { value: Function.CommentIds, label: "Comment IDs" },
			]}
			bind:selected={fun}
			expand={true}
		/>
		<div class="row">
			<TextField
				bind:text={subreddit}
				label="Subreddit"
				transform={(text) => text.replace(/^\/?r\//g, "")}
				getError={(text) => text.length == 0 || text.length >= 2 && text.match(/^[a-zA-Z0-9_\-]+$/) ? null : "Invalid subreddit"}
			/>
			<TextField
				bind:text={author}
				label="Author"
				transform={(text) => text.replace(/^\/?u(ser)?\//g, "")}
				getError={(text) => text.length == 0 || text.length >= 2 && text.match(/^[a-zA-Z0-9_\-\[\]]+$/) ? null : "Invalid author"}
			/>
		</div>
		<div class="row">
			<TextField
				bind:text={after}
				label="After (UTC)"
				type="datetime-local"
			/>
			<TextField
				bind:text={before}
				label="Before (UTC)"
				type="datetime-local"
			/>
		</div>
		<div class="row">
			<TextField
				bind:text={limit}
				label="Limit"
				type="number"
				min="1"
				max="100"
				getError={verifyLimit}
			/>
			<OptionSelector
				label="Date Sort"
				options={[
					{ value: "asc", label: "Ascending" },
					{ value: "desc", label: "Descending" },
				]}
				bind:selected={sort}
			/>
		</div>
		{#if fun === Function.PostsSearch || fun === Function.ThreadSearch}
			<div
				class="row"
				class:disabled-row={author.length == 0 && subreddit.length == 0}
			>
				<TextField
					bind:text={title}
					label="Title (only with 'Author' or 'Subreddit')"
				/>
				<TextField
					bind:text={selftext}
					label="Selftext (only with 'Author' or 'Subreddit')"
				/>
			</div>
			<TextField
				bind:text={url}
				label="URL (prefix match)"
				transform={(text) => text.replace(/^\/?u(ser)?\//g, "")}
				getError={(text) => text.length == 0 || text.length > 2 && !text.match(/^[a-zA-Z0-9_]+$/) ? null : "Invalid URL"}
			/>
			<div class="row">
				<OptionSelector
					label="NSFW"
					options={[
						{ value: true, label: "Yes" },
						{ value: false, label: "No" },
					]}
					bind:selected={over18}
					canDeselect={true}
				/>
				<OptionSelector
					label="Spoiler"
					options={[
						{ value: true, label: "Yes" },
						{ value: false, label: "No" },
					]}
					bind:selected={spoiler}
					canDeselect={true}
				/>
			</div>
		{:else if fun === Function.CommentsSearch}
			<div class="row">
				<TextField
					bind:text={linkId}
					label="Link ID"
					transform={(text) => text.replace(/^t3_/, "")}
					getError={(text) => text.length == 0 || text.match(/^[a-z0-9]+$/) ? null : "Invalid Base36 ID"}
				/>
				<TextField
					bind:text={parentId}
					label="Parent Comment ID"
					transform={(text) => text.replace(/^t1_/, "")}
					getError={(text) => text.length == 0 || text.match(/^[a-z0-9]+$/) ? null : "Invalid Base36 ID"}
				/>
			</div>
			<div
				class="row"
				class:disabled-row={linkId.length == 0 && parentId.length == 0 && author.length == 0 && subreddit.length == 0}
			>
				<TextField
					bind:text={body}
					label="Body (only with 'Author', 'Subreddit', 'Link ID' or 'Parent Comment ID')"
				/>
			</div>
		{/if}

		<div class="gap"></div>
	
		<div class="row">
			<div class="error">{error || ""}</div>
			<button
				class="settings-button"
				on:click={() => showSettings = !showSettings}
			>
				<img src={settingsSvg} alt="settings" />
			</button>
			{#if fun === Function.PostsSearch && posts?.length || fun === Function.CommentsSearch && comments?.length || fun === Function.ThreadSearch && posts?.length}
				<button
					class="submit-button secondary"
					on:click={download}
				>Download</button>
			{/if}
			<button
				class="submit-button"
				disabled={loading}
				on:click={search}
			>Search</button>
		</div>
		{#if error && error.toLowerCase().includes("timeout")}
			<p>If you're experiencing timeouts, try reducing the limit or narrowing down the time frame.</p>
		{/if}
	</div>

	{#if showSettings}
		<Preferences/>
	{/if}

	{#if currentData}
		<div class="pagination">
			<button
				class="submit-button"
				disabled={loading || previousHistory.length == 0}
				on:click={searchPrevious}
			>Previous</button>
			<button
				class="submit-button"
				disabled={loading}
				on:click={searchNext}
			>Next</button>
		</div>
	{/if}

	<div class="results" class:loading={loading}>
		{#if fun === Function.PostsSearch && posts !== null}
			{#if posts.length == 0}
				<p>Nothing found o_O</p>
			{/if}
			{#each posts as post (post.id)}
				<RedditPost data={post} newlyRemoved={newlyRemoved[post.id] || null} />
			{/each}
		{:else if fun === Function.CommentsSearch && comments !== null}
			{#if comments.length == 0}
				<p>Nothing found o_O</p>
			{/if}
			{#each comments as comment (comment.id)}
				<RedditComment data={comment} />
			{/each}
		{:else if fun === Function.ThreadSearch && posts !== null}
			{#if posts.length == 0}
				<p>No OP found.</p>
			{/if}
			{#each posts as post (post.id)}
				<RedditPost data={post} op={true} newlyRemoved={newlyRemoved[post.id] || null} />
			{/each}
			{#if comments !== null}
				{#if comments.length == 0}
					<p>No comments.</p>
				{/if}
				{#each comments as comment (comment.id)}
					<RedditComment data={comment} replies={replies} />
				{/each}
			{/if}
		{/if}
	</div>

	{#if currentData && currentData.length > 5}
		<div class="pagination">
			<button
				class="submit-button"
				disabled={loading || previousHistory.length == 0}
				on:click={searchPrevious}
			>Previous</button>
			<button
				class="submit-button"
				disabled={loading}
				on:click={searchNext}
			>Next</button>
		</div>
	{/if}
</div>

<style lang="scss">
	.search {
		display: flex;
		flex-direction: column;
		gap: 1rem;
		width: min(100%, 50rem);
		margin: 1rem auto;
		padding: 1rem;
		font-size: 1.2rem;
	}

	.parameters {
		display: flex;
		flex-direction: column;
		// gap: .5rem;
	}

	.row {
		display: flex;
		flex-direction: row;
		align-items: center;
		@media (max-width: 600px) {
			flex-direction: column;
			align-items: stretch;
		}
		gap: .5rem;
		width: 100%;
	
		> :global(*) {
			flex: 1;
		}
	}

	.title-row {
		justify-content: space-between;
		
		> :global(*) {
			flex: initial;
		}
	}

	.disabled-row {
		opacity: .5;
		pointer-events: none;
	}

	.gap {
		height: 1rem;
	}

	.submit-button {
		flex: 0;
		height: 2.5rem;
		line-height: 2.5rem;
		padding: 0 1rem;
		background: var(--bg-el2-color);
		border: 1px solid var(--border-color);
		border-radius: 2rem;
		font-size: 1.2rem;
		transition: background 0.25s ease;

		&.secondary {
			background: transparent;
		}

		&:disabled {
			opacity: .5;
			pointer-events: none;
		}

		&:hover {
			background: var(--switcher-bg-hover);
		}

		&:active {
			background: var(--switcher-bg-active);
		}
	}

	.settings-button {
		transition: background 0.25s ease;
		border-radius: 50%;
		height: 2.25rem;
		width: 2.25rem;
		display: flex;
		justify-content: center;
		align-items: center;
		flex: unset;

		&:hover {
			background: var(--transparent-btn-hover);
		}

		&:active {
			background: var(--transparent-btn-active);
		}

		img {
			width: 1.5rem;
			height: 1.5rem;
		}
	}

	.error {
		color: var(--error);
	}

	.pagination {
		display: flex;
		flex-direction: row;
		justify-content: space-between;
	}

	.results {
		display: flex;
		flex-direction: column;
		gap: 1rem;
		transition: opacity 0.25s ease;

		&.loading {
			opacity: .5;
			pointer-events: none;
		}
	}
</style>
