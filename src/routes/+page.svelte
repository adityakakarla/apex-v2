<script lang="ts">
    import { invoke } from "@tauri-apps/api/core";
    import { marked } from "marked";
    import { fly } from "svelte/transition";
    import { cubicOut } from "svelte/easing";

    // --- State ---
    let initialized = $state<boolean | null>(null);
    let courses = $state<string[]>([]);
    let selectedCourse = $state<string | null>(null);
    let sections = $state<string[]>([]);
    let selectedSection = $state<string | null>(null);
    let sectionContents = $state<string[]>([]);
    let selectedContent = $state<string | null>(null);
    let markdownHtml = $state<string>("");
    let quizPairs = $state<[string, string][]>([]);
    let progress = $state<string[]>([]);

    // Quiz state
    let quizIndex = $state(0);
    let showAnswer = $state(false);
    let quizResults = $state<boolean[]>([]);
    let quizDone = $state(false);

    // Copy command state
    let copied = $state(false);

    async function copyClaudeCommand() {
        const cmd: string = await invoke("get_claude_command");
        await invoke("copy_to_clipboard", { text: cmd });
        copied = true;
        setTimeout(() => (copied = false), 1800);
    }

    // --- Init ---
    async function boot() {
        initialized = await invoke("is_initialized");
        if (initialized) {
            courses = await invoke("get_courses");
            if (courses.length > 0) await selectCourse(courses[0]);
        }
    }

    async function setup() {
        const ok: boolean = await invoke("initialize_directory");
        if (ok) {
            initialized = true;
            courses = await invoke("get_courses");
        }
    }

    // --- Navigation ---
    async function selectCourse(course: string) {
        selectedCourse = course;
        selectedSection = null;
        selectedContent = null;
        markdownHtml = "";
        quizPairs = [];
        sections = await invoke("get_course_sections", { course });
        progress = await invoke("get_progress", { course });
    }

    async function selectSection(section: string) {
        if (!selectedCourse) return;
        selectedSection = section;
        selectedContent = null;
        markdownHtml = "";
        quizPairs = [];
        sectionContents = await invoke("get_course_section_contents", {
            course: selectedCourse,
            section,
        });
    }

    async function selectContent(content: string) {
        if (!selectedCourse || !selectedSection) return;
        selectedContent = content;
        quizIndex = 0;
        showAnswer = false;
        quizResults = [];
        quizDone = false;

        if (content.endsWith(".md")) {
            const raw: string = await invoke("get_markdown", {
                course: selectedCourse,
                section: selectedSection,
                content,
            });
            markdownHtml = await marked(raw);
            quizPairs = [];
        } else if (content.endsWith(".json")) {
            quizPairs = await invoke("get_quiz", {
                course: selectedCourse,
                section: selectedSection,
                content,
            });
            markdownHtml = "";
        }
    }

    async function completeContent() {
        if (!selectedCourse || !selectedSection || !selectedContent) return;
        await invoke("mark_complete", {
            course: selectedCourse,
            section: selectedSection,
            content: selectedContent,
        });
        progress = await invoke("get_progress", { course: selectedCourse });
    }

    function isComplete(section: string, content: string) {
        return progress.includes(`${section}/${content}`);
    }

    // Quiz controls
    function revealAnswer() {
        showAnswer = true;
    }

    function answerQuiz(correct: boolean) {
        quizResults = [...quizResults, correct];
        if (quizIndex + 1 >= quizPairs.length) {
            quizDone = true;
            completeContent();
        } else {
            quizIndex++;
            showAnswer = false;
        }
    }

    function restartQuiz() {
        quizIndex = 0;
        showAnswer = false;
        quizResults = [];
        quizDone = false;
    }

    function fileLabel(name: string) {
        const base = name.replace(/\.(md|json)$/, "").replace(/[-_]/g, " ");
        return base.charAt(0).toUpperCase() + base.slice(1);
    }

    function isJson(name: string) {
        return name.endsWith(".json");
    }

    $effect(() => {
        boot();
    });
</script>

{#if initialized === null}
    <div class="splash">
        <div class="splash-inner">
            <span class="splash-logo">apex</span>
            <p class="splash-sub">loading…</p>
        </div>
    </div>
{:else if !initialized}
    <div class="splash">
        <div class="splash-inner">
            <span class="splash-logo">apex</span>
            <p class="splash-sub">your personal course library</p>
            <button class="btn-primary" onclick={setup}
                >Initialize workspace</button
            >
        </div>
    </div>
{:else}
    <div class="shell">
        <!-- Top header -->
        <header class="topbar">
            <span class="wordmark">apex</span>
            <button
                class="btn-copy-cmd"
                class:copied
                onclick={copyClaudeCommand}
            >
                {copied ? "Copied!" : "Copy Claude Command"}
            </button>
        </header>

        <div class="shell-body">
            <!-- Sidebar: courses -->
            <aside class="sidebar-courses">
                <nav class="course-list">
                    {#each courses as course}
                        {@const done =
                            sections.length > 0 && selectedCourse === course
                                ? sections.every(
                                      (s) =>
                                          sectionContents.length === 0 ||
                                          sectionContents.every((c) =>
                                              isComplete(s, c),
                                          ),
                                  )
                                : false}
                        <button
                            class="course-item"
                            class:active={selectedCourse === course}
                            onclick={() => selectCourse(course)}
                        >
                            <span class="course-dot" class:done></span>
                            <span class="course-name">{fileLabel(course)}</span>
                        </button>
                    {/each}
                    {#if courses.length === 0}
                        <p class="empty-hint">
                            No courses yet.<br />Use Claude to generate content.
                        </p>
                    {/if}
                </nav>
            </aside>

            <!-- Sidebar: sections -->
            {#if selectedCourse}
                <aside class="sidebar-sections">
                    <nav class="section-list">
                        {#each sections as section}
                            <button
                                class="section-item"
                                class:active={selectedSection === section}
                                onclick={() => selectSection(section)}
                            >
                                <span
                                    class="section-chevron"
                                    class:open={selectedSection === section}
                                    >▶</span
                                >
                                {fileLabel(section)}
                            </button>
                            {#if selectedSection === section && sectionContents.length > 0}
                                <div class="content-list">
                                    {#each sectionContents as content}
                                        {@const done = isComplete(
                                            section,
                                            content,
                                        )}
                                        <button
                                            class="content-item"
                                            class:active={selectedContent ===
                                                content}
                                            class:done
                                            onclick={() =>
                                                selectContent(content)}
                                        >
                                            <span
                                                class="content-type-badge"
                                                class:quiz={isJson(content)}
                                            >
                                                {isJson(content) ? "Q" : "R"}
                                            </span>
                                            <span>{fileLabel(content)}</span>
                                            {#if done}<span class="check"
                                                    >✓</span
                                                >{/if}
                                        </button>
                                    {/each}
                                </div>
                            {/if}
                        {/each}
                    </nav>
                </aside>
            {/if}

            <!-- Main content area -->
            <main class="main">
                {#if !selectedContent}
                    <div class="empty-state">
                        {#if !selectedCourse}
                            <div class="empty-state-text">
                                <span class="empty-big">Select a course</span>
                                <span class="empty-small"
                                    >to begin learning</span
                                >
                            </div>
                        {:else if !selectedSection}
                            <div class="empty-state-text">
                                <span class="empty-big">Pick a section</span>
                                <span class="empty-small"
                                    >from {fileLabel(selectedCourse)}</span
                                >
                            </div>
                        {:else}
                            <div class="empty-state-text">
                                <span class="empty-big">Choose content</span>
                                <span class="empty-small"
                                    >to read or quiz yourself</span
                                >
                            </div>
                        {/if}
                    </div>
                {:else if markdownHtml}
                    <article class="prose-wrap">
                        <div class="prose-header">
                            <span class="prose-breadcrumb"
                                >{fileLabel(selectedCourse ?? "")} / {fileLabel(
                                    selectedSection ?? "",
                                )}</span
                            >
                            <h1 class="prose-title">
                                {fileLabel(selectedContent)}
                            </h1>
                        </div>
                        <div class="prose">{@html markdownHtml}</div>
                        {#if !isComplete(selectedSection ?? "", selectedContent)}
                            <button
                                class="btn-complete"
                                onclick={completeContent}
                            >
                                Mark as complete
                            </button>
                        {:else}
                            <div class="complete-badge">Completed ✓</div>
                        {/if}
                    </article>
                {:else if quizPairs.length > 0}
                    <div class="quiz-wrap">
                        <div class="quiz-header">
                            <span class="prose-breadcrumb"
                                >{fileLabel(selectedCourse ?? "")} / {fileLabel(
                                    selectedSection ?? "",
                                )}</span
                            >
                            <h1 class="prose-title">
                                {fileLabel(selectedContent)}
                            </h1>
                        </div>

                        {#if quizDone}
                            <div class="quiz-done">
                                <div class="quiz-score">
                                    {quizResults.filter(Boolean).length}<span
                                        class="quiz-score-denom"
                                        >/{quizPairs.length}</span
                                    >
                                </div>
                                <p class="quiz-score-label">
                                    {quizResults.filter(Boolean).length ===
                                    quizPairs.length
                                        ? "Perfect score!"
                                        : quizResults.filter(Boolean).length >=
                                            quizPairs.length / 2
                                          ? "Good effort"
                                          : "Keep practicing"}
                                </p>
                                <div class="quiz-pips">
                                    {#each quizResults as r}
                                        <span
                                            class="quiz-pip"
                                            class:correct={r}
                                            class:wrong={!r}
                                        ></span>
                                    {/each}
                                </div>
                                <button
                                    class="btn-primary"
                                    onclick={restartQuiz}>Retry quiz</button
                                >
                            </div>
                        {:else}
                            <div class="quiz-progress-bar">
                                <div
                                    class="quiz-progress-fill"
                                    style="width: {(quizIndex /
                                        quizPairs.length) *
                                        100}%"
                                ></div>
                            </div>
                            <span class="quiz-counter"
                                >{quizIndex + 1} / {quizPairs.length}</span
                            >

                            <div class="card-track">
                                {#key quizIndex}
                                    <div
                                        class="card-scene"
                                        in:fly={{
                                            x: 48,
                                            duration: 320,
                                            easing: cubicOut,
                                        }}
                                        out:fly={{
                                            x: -48,
                                            duration: 200,
                                            easing: cubicOut,
                                        }}
                                    >
                                        <div
                                            class="card"
                                            class:flipped={showAnswer}
                                        >
                                            <div class="card-face card-front">
                                                <span class="card-label"
                                                    >Question</span
                                                >
                                                <p class="card-text">
                                                    {quizPairs[quizIndex][0]}
                                                </p>
                                                <button
                                                    class="btn-reveal"
                                                    onclick={revealAnswer}
                                                    >Reveal answer</button
                                                >
                                            </div>
                                            <div class="card-face card-back">
                                                <span class="card-label"
                                                    >Answer</span
                                                >
                                                <p class="card-text">
                                                    {quizPairs[quizIndex][1]}
                                                </p>
                                                <div class="quiz-actions">
                                                    <button
                                                        class="btn-wrong"
                                                        onclick={() =>
                                                            answerQuiz(false)}
                                                    >
                                                        ✗ Missed it
                                                    </button>
                                                    <button
                                                        class="btn-correct"
                                                        onclick={() =>
                                                            answerQuiz(true)}
                                                    >
                                                        ✓ Got it
                                                    </button>
                                                </div>
                                            </div>
                                        </div>
                                    </div>
                                {/key}
                            </div>
                        {/if}
                    </div>
                {:else}
                    <div class="empty-state">
                        <span class="empty-small">No content found.</span>
                    </div>
                {/if}
            </main>
        </div>
        <!-- shell-body -->
    </div>
{/if}

<style>
    :global(*) {
        box-sizing: border-box;
        margin: 0;
        padding: 0;
    }
    :global(body) {
        background: #0b0c10;
        color: #ede8e0;
        font-family: "DM Mono", monospace;
        font-size: 14px;
        height: 100vh;
        overflow: hidden;
    }

    /* --- Splash --- */
    .splash {
        height: 100vh;
        display: flex;
        align-items: center;
        justify-content: center;
        background: #0b0c10;
    }
    .splash-inner {
        display: flex;
        flex-direction: column;
        align-items: center;
        gap: 16px;
    }
    .splash-logo {
        font-family: "Fraunces", serif;
        font-size: 64px;
        font-weight: 300;
        letter-spacing: -2px;
        color: #f0ebe3;
    }
    .splash-sub {
        color: #b8b0a8;
        font-size: 13px;
        letter-spacing: 0.04em;
    }

    /* --- Shell layout --- */
    .shell {
        display: flex;
        flex-direction: column;
        height: 100vh;
        overflow: hidden;
    }
    .topbar {
        display: flex;
        align-items: center;
        justify-content: space-between;
        padding: 0 20px;
        height: 48px;
        flex-shrink: 0;
        border-bottom: 2px solid #2e3140;
        background: #0e0f14;
    }
    .shell-body {
        display: flex;
        flex: 1;
        overflow: hidden;
    }

    /* --- Sidebar shared --- */
    .sidebar-courses,
    .sidebar-sections {
        display: flex;
        flex-direction: column;
        border-right: 2px solid #2e3140;
        overflow-y: auto;
        flex-shrink: 0;
    }
    .sidebar-courses {
        width: 200px;
        background: #0e0f14;
    }
    .sidebar-sections {
        width: 220px;
        background: #111318;
    }
    .wordmark {
        font-family: "Fraunces", serif;
        font-size: 22px;
        font-weight: 300;
        letter-spacing: -0.5px;
        color: #e8a87c;
    }

    /* --- Course list --- */
    .course-list {
        padding: 8px 0;
        flex: 1;
    }
    .course-item {
        display: flex;
        align-items: center;
        gap: 10px;
        width: 100%;
        padding: 8px 16px;
        background: none;
        border: none;
        cursor: pointer;
        color: #c0b8b0;
        font-family: "DM Mono", monospace;
        font-size: 12px;
        letter-spacing: 0.02em;
        text-align: left;
        transition: color 0.15s;
    }
    .course-item:hover {
        color: #e0dbd3;
    }
    .course-item.active {
        color: #f5f0e8;
    }
    .course-dot {
        width: 6px;
        height: 6px;
        border-radius: 50%;
        background: #2a2c34;
        border: 1px solid #3a3c44;
        flex-shrink: 0;
        transition: background 0.2s;
    }
    .course-dot.done {
        background: #e8a87c;
        border-color: #e8a87c;
    }
    .course-item.active .course-dot {
        background: #e8a87c;
        border-color: #e8a87c;
    }
    .course-name {
        overflow: hidden;
        text-overflow: ellipsis;
        white-space: nowrap;
    }
    .empty-hint {
        padding: 16px;
        color: #9a9caa;
        font-size: 12px;
        line-height: 1.6;
    }

    /* --- Section list --- */
    .section-list {
        padding: 8px 0;
    }
    .section-item {
        display: flex;
        align-items: center;
        gap: 7px;
        width: 100%;
        padding: 7px 16px;
        background: none;
        border: none;
        cursor: pointer;
        color: #bab4ac;
        font-family: "DM Mono", monospace;
        font-size: 12px;
        text-align: left;
        transition: color 0.15s;
        white-space: nowrap;
        overflow: hidden;
        text-overflow: ellipsis;
    }
    .section-item:hover {
        color: #e0dbd3;
    }
    .section-item.active {
        color: #e8a87c;
    }
    .section-chevron {
        font-size: 8px;
        flex-shrink: 0;
        transition: transform 0.15s;
        color: #4a4c58;
    }
    .section-chevron.open {
        transform: rotate(90deg);
        color: #e8a87c;
    }

    /* --- Content list --- */
    .content-list {
        padding: 2px 0 6px;
        border-left: 2px solid #2e3140;
        margin-left: 23px;
    }
    .content-item {
        display: flex;
        align-items: center;
        gap: 8px;
        width: 100%;
        padding: 5px 12px;
        background: none;
        border: none;
        cursor: pointer;
        color: #bab4ac;
        font-family: "DM Mono", monospace;
        font-size: 12px;
        text-align: left;
        transition: color 0.15s;
    }
    .content-item:hover {
        color: #e0dbd3;
    }
    .content-item.active {
        color: #f5f0e8;
    }
    .content-item.done {
        color: #7ab87a;
    }
    .content-item.done.active {
        color: #9ee09e;
    }
    .content-type-badge {
        font-size: 9px;
        padding: 1px 4px;
        border-radius: 2px;
        background: #1c1e24;
        color: #a8a4a0;
        flex-shrink: 0;
    }
    .content-type-badge.quiz {
        background: #1e1c24;
        color: #b8b4f0;
    }
    .check {
        margin-left: auto;
        color: #7ab87a;
        font-size: 11px;
    }

    /* --- Main area --- */
    .main {
        flex: 1;
        overflow-y: auto;
        background: #0b0c10;
    }
    .btn-copy-cmd {
        padding: 6px 14px;
        background: none;
        border: 1px solid #3a3c48;
        border-radius: 4px;
        color: #b0aca8;
        font-family: "DM Mono", monospace;
        font-size: 11px;
        cursor: pointer;
        transition:
            border-color 0.15s,
            color 0.15s;
        letter-spacing: 0.02em;
    }
    .btn-copy-cmd:hover {
        border-color: #e8a87c;
        color: #e8a87c;
    }
    .btn-copy-cmd.copied {
        border-color: #5a9a5a;
        color: #7ac87a;
    }

    /* --- Empty state --- */
    .empty-state {
        height: 100%;
        display: flex;
        align-items: center;
        justify-content: center;
    }
    .empty-state-text {
        display: flex;
        flex-direction: column;
        align-items: center;
        gap: 8px;
    }
    .empty-big {
        font-family: "Fraunces", serif;
        font-size: 28px;
        font-weight: 300;
        color: #7a7c8a;
    }
    .empty-small {
        font-size: 12px;
        color: #8a8c9a;
    }

    /* --- Prose --- */
    .prose-wrap {
        max-width: 680px;
        margin: 0 auto;
        padding: 48px 40px 80px;
    }
    .prose-header {
        margin-bottom: 32px;
    }
    .prose-breadcrumb {
        font-size: 11px;
        color: #8a8c9a;
        letter-spacing: 0.06em;
        text-transform: uppercase;
    }
    .prose-title {
        font-family: "Fraunces", serif;
        font-size: 36px;
        font-weight: 300;
        color: #f0ebe3;
        letter-spacing: -0.5px;
        margin-top: 8px;
        line-height: 1.2;
    }
    .prose :global(h1),
    .prose :global(h2),
    .prose :global(h3) {
        font-family: "Fraunces", serif;
        font-weight: 400;
        color: #f0ebe3;
        margin: 2em 0 0.6em;
        line-height: 1.25;
    }
    .prose :global(h1) {
        font-size: 26px;
    }
    .prose :global(h2) {
        font-size: 20px;
    }
    .prose :global(h3) {
        font-size: 16px;
    }
    .prose :global(p) {
        line-height: 1.75;
        color: #dcd6ce;
        margin: 0 0 1em;
        font-size: 14px;
    }
    .prose :global(code) {
        background: #14161c;
        color: #e8a87c;
        padding: 2px 6px;
        border-radius: 3px;
        font-family: "DM Mono", monospace;
        font-size: 13px;
    }
    .prose :global(pre) {
        background: #14161c;
        border: 1px solid #1c1e24;
        border-radius: 6px;
        padding: 16px 20px;
        overflow-x: auto;
        margin: 1.2em 0;
    }
    .prose :global(pre code) {
        background: none;
        padding: 0;
        color: #c4beb7;
    }
    .prose :global(ul),
    .prose :global(ol) {
        padding-left: 1.4em;
        color: #dcd6ce;
        margin: 0 0 1em;
        line-height: 1.75;
    }
    .prose :global(li) {
        margin: 0.3em 0;
    }
    .prose :global(blockquote) {
        border-left: 3px solid #e8a87c;
        padding-left: 16px;
        color: #bcb6ae;
        font-style: italic;
        margin: 1.2em 0;
    }
    .prose :global(hr) {
        border: none;
        border-top: 1px solid #1c1e24;
        margin: 2em 0;
    }
    .prose :global(a) {
        color: #e8a87c;
        text-decoration: none;
    }
    .prose :global(a:hover) {
        text-decoration: underline;
    }
    .prose :global(strong) {
        color: #e2ddd6;
        font-weight: 500;
    }

    /* --- Buttons --- */
    .btn-primary {
        padding: 10px 24px;
        background: #e8a87c;
        color: #0b0c10;
        border: none;
        border-radius: 4px;
        font-family: "DM Mono", monospace;
        font-size: 13px;
        cursor: pointer;
        transition: opacity 0.15s;
    }
    .btn-primary:hover {
        opacity: 0.85;
    }
    .btn-complete {
        margin-top: 40px;
        padding: 9px 20px;
        background: none;
        border: 1px solid #3a3c48;
        color: #b8b4ac;
        border-radius: 4px;
        font-family: "DM Mono", monospace;
        font-size: 12px;
        cursor: pointer;
        transition:
            border-color 0.15s,
            color 0.15s;
    }
    .btn-complete:hover {
        border-color: #7ab87a;
        color: #9ee09e;
    }
    .complete-badge {
        margin-top: 40px;
        font-size: 12px;
        color: #8ed08e;
    }

    /* --- Quiz --- */
    .quiz-wrap {
        max-width: 600px;
        margin: 0 auto;
        padding: 48px 40px 80px;
    }
    .quiz-header {
        margin-bottom: 32px;
    }
    .quiz-progress-bar {
        height: 2px;
        background: #1c1e24;
        border-radius: 1px;
        margin-bottom: 8px;
        overflow: hidden;
    }
    .quiz-progress-fill {
        height: 100%;
        background: #e8a87c;
        border-radius: 1px;
        transition: width 0.3s ease;
    }
    .quiz-counter {
        font-size: 11px;
        color: #8a8c9a;
        display: block;
        margin-bottom: 28px;
    }
    .card-track {
        position: relative;
        min-height: 260px;
        overflow: hidden;
    }
    .card-scene {
        perspective: 1200px;
        position: absolute;
        inset: 0;
    }
    .card {
        display: grid;
        min-height: 260px;
        height: 100%;
        transform-style: preserve-3d;
        transition: transform 0.55s cubic-bezier(0.4, 0, 0.2, 1);
        border-radius: 8px;
    }
    .card.flipped {
        transform: rotateY(180deg);
    }
    .card.flipped .card-front {
        pointer-events: none;
    }
    .card-face {
        grid-area: 1 / 1;
        backface-visibility: hidden;
        -webkit-backface-visibility: hidden;
        display: flex;
        flex-direction: column;
        gap: 16px;
        padding: 36px 32px 28px;
        background: #111318;
        border: 1px solid #1c1e24;
        border-radius: 8px;
    }
    .card-back {
        transform: rotateY(180deg);
        border-color: #2a2c3a;
        background: #12121a;
    }
    .card-label {
        font-size: 10px;
        letter-spacing: 0.12em;
        text-transform: uppercase;
        color: #8a8c9a;
    }
    .card-back .card-label {
        color: #9898c8;
    }
    .card-text {
        font-family: "Fraunces", serif;
        font-size: 22px;
        font-weight: 300;
        color: #f0ebe3;
        line-height: 1.45;
        flex: 1;
    }
    .btn-reveal {
        align-self: flex-start;
        margin-top: auto;
        padding: 8px 18px;
        background: none;
        border: 1px solid #3a3c48;
        border-radius: 4px;
        color: #b8b4ac;
        font-family: "DM Mono", monospace;
        font-size: 12px;
        cursor: pointer;
        transition:
            border-color 0.15s,
            color 0.15s;
    }
    .btn-reveal:hover {
        border-color: #e8a87c;
        color: #e8a87c;
    }
    .quiz-actions {
        display: flex;
        gap: 12px;
        margin-top: auto;
    }
    .btn-wrong,
    .btn-correct {
        flex: 1;
        padding: 10px;
        border-radius: 4px;
        font-family: "DM Mono", monospace;
        font-size: 12px;
        cursor: pointer;
        border: none;
        transition: opacity 0.15s;
    }
    .btn-wrong {
        background: #221616;
        color: #e08080;
    }
    .btn-wrong:hover {
        opacity: 0.8;
    }
    .btn-correct {
        background: #162216;
        color: #80d080;
    }
    .btn-correct:hover {
        opacity: 0.8;
    }
    .quiz-done {
        display: flex;
        flex-direction: column;
        align-items: center;
        gap: 16px;
        padding: 48px 0;
    }
    .quiz-score {
        font-family: "Fraunces", serif;
        font-size: 72px;
        font-weight: 300;
        color: #f0ebe3;
        line-height: 1;
    }
    .quiz-score-denom {
        font-size: 36px;
        color: #8a8c9a;
    }
    .quiz-score-label {
        font-size: 13px;
        color: #b8b4ac;
    }
    .quiz-pips {
        display: flex;
        gap: 6px;
        margin: 8px 0;
    }
    .quiz-pip {
        width: 8px;
        height: 8px;
        border-radius: 50%;
    }
    .quiz-pip.correct {
        background: #5a9a5a;
    }
    .quiz-pip.wrong {
        background: #9a5a5a;
    }
</style>
