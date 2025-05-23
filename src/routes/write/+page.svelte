<script lang="ts">
	import { botName } from '$lib/config'
	import type { ChatResponse } from '../api/write/+server'
	import { onMount } from 'svelte'

	// Define local Message type to include 'progress' role and complete flag
	type Message = {
		role: 'user' | 'assistant' | 'system' | 'progress'
		content: string
		complete?: boolean // Flag to indicate if processing is complete
	}

	// Define field section structure
	interface FieldSection {
		title: string
		subsections: FieldSubsection[]
	}

	interface FieldSubsection {
		title: string
		questions: string[]
	}

	// Use a unique localStorage key to avoid conflicts with other pages
	const STORAGE_KEY = 'email_writer_messages'

	let messages: Message[] =
		typeof localStorage !== 'undefined' ? JSON.parse(localStorage.getItem(STORAGE_KEY) || '[]') : []

	// Array for form
	let input_arr = new Array<string>(35)
	let loading = false
	let apiAvailable = true // Default to true, will be updated after first API call
	const maxMessages = 20

	// Organizing the form questions into sections and subsections
	const formSections_Target: FieldSection[] = [
		{
			title: 'Personal Context',
			subsections: [
				{
					title: 'Professional Title and Status',
					questions: [
						'Appropriate salutation and level of formality',
						'Understanding their role and potential authority',
						'How they might perceive communication'
					]
				},
				{
					title: 'Prior Interactions/Background',
					questions: [
						'Known positions or statements',
						'Recent actions or achievements relevant to your message'
					]
				},
				{
					title: 'Communication Preferences',
					questions: ['Preferred communication style']
				},
				{
					title: 'Potential Motivations',
					questions: ['What might incentivize them to act', 'Their likely concerns or interests'] /*
					questions: [
						'What might incentivize them to act',
						'Their likely concerns or interests',
						'Potential alignment with your request'
					]*/
				}
			]
		},
		{
			title: 'Psychological Considerations',
			subsections: [
				{
					title: 'Emotional Landscape',
					questions: [
						'Potential receptiveness to your message',
						'Avoiding triggers that might create resistance'
					] /*
					questions: [
						'Current potential mood or state of mind',
						'Potential receptiveness to your message',
						'Avoiding triggers that might create resistance'
					]*/
				},
				{
					title: 'Perspective Alignment',
					questions: [
						'Shared values or goals',
						'Ways to frame your message to resonate with them',
						'Avoiding controversial or dismissive language'
					]
				}
			]
		}
	]

	// Flatten the questions array for accessing by index
	const paragraphText_Target: string[] = []
	formSections_Target.forEach((section) => {
		section.subsections.forEach((subsection) => {
			subsection.questions.forEach((question) => {
				paragraphText_Target.push(question)
			})
		})
	})

	const formSections_Research: FieldSection[] = [
		{
			title: 'Finding a target',
			subsections: [
				{
					title: "Specify what sort of target you're looking for, and where.",
					questions: [
						'If you have certain institutions in mind, mention those. Otherwise, your local representative could be a good place to start.'
					]
				}
			]
		}
	]

	// Flatten the questions array for accessing by index
	const paragraphText_Research: string[] = []
	formSections_Research.forEach((section) => {
		section.subsections.forEach((subsection) => {
			subsection.questions.forEach((question) => {
				paragraphText_Research.push(question)
			})
		})
	})

	const formSections_MessageDetails: FieldSection[] = [
		{
			title: 'The Message',
			subsections: [
				{
					title: 'Logical Structure',
					questions: [
						'Chronological flow',
						'Cause-and-effect relationships',
						'Anticipated questions or objections'
					]
				}
			]
		},
		{
			title: 'Practical Elements',
			subsections: [
				{
					title: 'Timeframe',
					questions: ['Urgency of the request', 'Specific deadlines'] /*
					questions: [
						'Urgency of the request',
						'Specific deadlines',
						'Expected timeline for response or action'
					]*/
				},
				{
					title: 'Supplementary Information',
					questions: ['Attachments needed', 'Links to additional resources']
					//questions: ['Attachments needed', 'Links to additional resources', 'Reference materials']
				}
			]
		},
		{
			title: 'Communication Strategy',
			subsections: [
				{
					title: 'Tone Calibration',
					questions: [
						"Matching recipient's communication style",
						'Balancing professionalism and approachability'
					]
				},
				{
					title: 'Persuasion Techniques',
					questions: [
						'Highlighting benefits',
						'Creating a sense of urgency',
						'Making the action feel achievable'
					]
				}
			]
		}
	]

	// Flatten the questions array for accessing by index
	const paragraphText_MessageDetails: string[] = []
	formSections_Research.forEach((section) => {
		section.subsections.forEach((subsection) => {
			subsection.questions.forEach((question) => {
				paragraphText_MessageDetails.push(question)
			})
		})
	})

	const formSections_Message: FieldSection[] = [
		{
			title: 'What is your Message?',
			subsections: [
				{
					title: 'Content Requirements',
					questions: ['Specific outcome desired', 'Concrete action requested'] /*
					questions: [
						'Clear, singular objective',
						'Specific outcome desired',
						'Concrete action requested'
					]*/
				},
				{
					title: 'Supporting Evidence',
					questions: [
						'Relevant facts',
						'Context for the request',
						'Potential impact or consequences'
					]
				}
			]
		}
	]

	// Flatten the questions array for accessing by index
	const paragraphText_Message: string[] = []
	formSections_Message.forEach((section) => {
		section.subsections.forEach((subsection) => {
			subsection.questions.forEach((question) => {
				paragraphText_Message.push(question)
			})
		})
	})

	function clear_arr(arr: string[]) {
		for (var i in arr) {
			arr[i] = ''
		}
	}

	function clear() {
		messages = []
		localStorage.setItem(STORAGE_KEY, JSON.stringify(messages))
	}

	function copy() {
		const role = (message: Message) => (message.role === 'user' ? 'You' : 'Writer')
		const text = messages.map((message) => `${role(message)}:\n${message.content}`).join('\n\n')
		navigator.clipboard.writeText(text)
		window.alert(
			"Copied to clipboard\n\nDisclaimer: Check all text content carefully before sending anything!\n\nYou are the one sending the email and expressing an opinion. You've simply had some assistance in writing it."
		)
	}

	// Magic test function that populates three specific fields and starts the process - for demo purposes
	function runTest() {
		// Clear any existing chat
		clear()

		// Clear all input fields
		clear_arr(input_arr)

		// Find index for specific fields based on their question text
		const roleAuthorityIndex = paragraphText_Target.findIndex(
			(q) => q === 'Understanding their role and potential authority'
		)
		const objectiveIndex = paragraphText_Target.findIndex((q) => q === 'Concrete action requested')
		const outcomeIndex = paragraphText_Target.findIndex((q) => q === 'Specific outcome desired')

		input_arr[roleAuthorityIndex] = 'You are writing for a child of about thirteen years old.'
		input_arr[objectiveIndex] =
			'Child understands that the current fast worldwide race to build AGI before we know how to make it safe could mean everybody dies. It sounds like science fiction, but experts are increasingly concerned. Of course the grown-ups should do something about this, but because it can be hard for them to admit to themselves that those they love could be in so much danger, then even once they hear about the problem they find reasons not to believe or act. Being sincere, kind and persistent with them that you need protecting can help everybody in the world.'
		input_arr[outcomeIndex] =
			"The child isn't too scared, but thinks this is a very important problem all their friends, family and teachers should know about and be actively working to solve."

		sendMessage()
	}

	async function sendMessage() {
		let input = ''
		for (var i in paragraphText_Target) {
			input = input + paragraphText_Target[i] + ':\n' + input_arr[i] + '\n\n'
		}
		messages = [...messages, { content: input, role: 'user' }]

		clear_arr(input_arr)
		loading = true

		try {
			// First request - get initial progress message (no stateToken)
			const initialResponse = await fetch('api/write', {
				method: 'POST',
				headers: {
					'Content-Type': 'application/json'
				},
				body: JSON.stringify([{ content: input, role: 'user' }])
			})

			const initialData = await initialResponse.json()

			// Add server-generated progress message with complete flag
			messages = [
				...messages,
				{
					content: initialData.progressString,
					role: 'progress',
					complete: initialData.complete // Track completion status
				}
			]

			// Scroll to progress message
			setTimeout(() => {
				const progressMessage = document.querySelector('.message.progress')
				if (progressMessage) {
					progressMessage.scrollIntoView({ behavior: 'smooth', block: 'center' })
				}
			}, 100)

			// Continue with the normal process, but pass the stateToken
			await processSteps(null, initialData.stateToken)
		} catch (error) {
			console.error('Error calling email API:', error)
			messages = [
				...messages,
				{
					content:
						'Sorry, there was an error generating your email. Please try again later or contact support.',
					role: 'assistant'
				}
			]
		} finally {
			loading = false
		}
	}

	async function processSteps(inputMessages: Message[] | null, stateToken = null) {
		// Set loading indicator
		loading = true

		try {
			// Prepare the request body
			const requestBody = stateToken ? { stateToken } : inputMessages

			// Make the API call
			const response = await fetch('api/write', {
				method: 'POST',
				headers: {
					'Content-Type': 'application/json'
				},
				body: JSON.stringify(requestBody)
			})

			// Process the response
			const data = await response.json()

			// Update API availability
			apiAvailable = data.apiAvailable !== false

			// Show progress if available
			if (data.progressString) {
				// Find existing progress message or create one
				const progressIndex = messages.findIndex((m) => m.role === 'progress')
				if (progressIndex >= 0) {
					// Update existing progress message
					messages[progressIndex].content = data.progressString
					messages[progressIndex].complete = data.complete // Update complete flag
				} else {
					// Add new progress message
					messages = [
						...messages,
						{
							content: data.progressString,
							role: 'progress',
							complete: data.complete // Add complete flag
						}
					]
				}
			}

			// Update the UI with the current state of the email
			if (data.response) {
				// Find existing assistant message or create one
				const assistantIndex = messages.findIndex((m) => m.role === 'assistant')
				if (assistantIndex >= 0) {
					// Update existing assistant message
					messages[assistantIndex].content = data.response
				} else {
					// Add new assistant message
					messages = [{ content: data.response, role: 'assistant' }, ...messages]
				}
			}

			// Update form fields if we have information from the research step
			if (data.information) {
				// Parse the information string to extract field values
				const lines = data.information.split('\n')
				let currentField = -1

				for (let line of lines) {
					// Look for field headers matching our paragraphText_Target array
					const fieldIndex = paragraphText_Target.findIndex((text) =>
						line.trim().startsWith(text + ':')
					)

					if (fieldIndex >= 0) {
						currentField = fieldIndex
					} else if (currentField >= 0 && line.trim()) {
						// Only update when there's actual content and the field is empty
						const lineContent = line.trim()
						if (lineContent && (!input_arr[currentField] || input_arr[currentField] === '')) {
							input_arr[currentField] = lineContent
						}
					}
				}
			}

			// Save messages to localStorage
			localStorage.setItem(STORAGE_KEY, JSON.stringify(messages))

			// If not complete, continue with the next step
			if (!data.complete && data.stateToken) {
				// Short delay to allow UI to update
				await new Promise((resolve) => setTimeout(resolve, 100))

				// Process the next step
				await processSteps(null, data.stateToken)
			} else {
				// Process is complete, update UI
				loading = false
			}
		} catch (error) {
			console.error('Error in step processing:', error)
			loading = false
			throw error
		}
	}

	onMount(async () => {
		// Check API availability on component mount
		try {
			const response = await fetch('api/write')
			const data = await response.json()
			apiAvailable = !!data.apiAvailable
		} catch (error) {
			console.error('Error checking API availability:', error)
			apiAvailable = false
		}
	})

	function handleKeyDown(event: KeyboardEvent) {
		/*
		if (event.key === 'Enter') {
			event.preventDefault()
			sendMessage()
			clear_arr(input_arr);
		}
		*/
	}

	// Function to get the index of a question across all sections
	function getQuestionIndex(question: string): number {
		return paragraphText_Target.findIndex((text) => text === question)
	}

	// FORM FUNCTIONS //

	// Add these variables for form toggling
	let activeForm = 'form1' // Default active form

	// Function to set the active form
	function setActiveForm(formId) {
		activeForm = formId
		console.log(formId)
	}

	// UNTESTED AND GENERATED BY AI
	// Create separate arrays for each form's inputs
	//let input_arr = Array(formSections_Target.flatMap(s => s.subsections.flatMap(ss => ss.questions)).length).fill('');
	let form2_input_arr = Array(
		formSections_Target.flatMap((s) => s.subsections.flatMap((ss) => ss.questions)).length
	).fill('')
	let form3_input_arr = Array(
		formSections_Target.flatMap((s) => s.subsections.flatMap((ss) => ss.questions)).length
	).fill('')
	let form4_input_arr = Array(
		formSections_Target.flatMap((s) => s.subsections.flatMap((ss) => ss.questions)).length
	).fill('')

	// Top of the page
	const title = `Write Email Content`
	const description = `This (beta!) webpage lets you write email content (with LLM assistance.)`
	const warning = `This feature is currently in beta testing. It is accessible, but the URL isn't advertised. Pause AI folk testing, contact the software team on Discord to report issues.`
</script>

<svelte:head>
	<title>{title}</title>
	<meta name="robots" content="noindex, nofollow" />
	<meta property="og:title" content={title} />
	<meta property="og:description" content={description} />
</svelte:head>

<main>
	<div class="header-section">
		<h1>{title}</h1>
		<div class="warning-box">
			<p class="warning">{warning}</p>
			{#if !apiAvailable}
				<p class="warning error">
					⚠️ API key not available. This feature is currently disabled. Please contact the site
					administrator.
				</p>
			{/if}
		</div>
		<p>
			"Answer questions / fill fields after researching your target. Undefined fields will be
			auto-filled. Check the generated email content carefully, as we're bound to make some
			mistakes!"
		</p>
		<p class="notes">
			The real user interface for entering inputs will surely be refined. For now, scan over the
			thirty(!) imperfectly structured input fields at the end of the page, and fill in the ones
			that seem the most important. See you back here when done!
		</p>
		<p class="notes">
			You can then ask to write content. The AI assistant will auto-fill any fields you didn't
			define, based on the ones you did, then proceed to craft an email over a number of steps. The
			UX for this part is closer to something we would launch but please give the software team
			further feedback!
		</p>
		<p class="notes">
			This is currently a very general writer. If you want it to write to your dad about puppies or
			to Trump about how we need to accelerate AI development, it will. We would probably give it
			more defaults and impose some restrictions in a truly public version.
		</p>
		<p>
			For a very quick demo, you can use the button below - it fills in just three fields with
			particular hardcoded values, and starts writing content.
		</p>

		<!-- Form toggle bar -->
		<div class="control-buttons">
			<button
				class="button {activeForm === 'form1' ? 'active' : ''}"
				on:click={() => setActiveForm('form1')}
			>
				Finding A Target
			</button>
			<button
				class="button {activeForm === 'form2' ? 'active' : ''}"
				on:click={() => setActiveForm('form2')}
			>
				Personal Context
			</button>
			<button
				class="button {activeForm === 'form3' ? 'active' : ''}"
				on:click={() => setActiveForm('form3')}
			>
				The Message
			</button>
			<button
				class="button {activeForm === 'form4' ? 'active' : ''}"
				on:click={() => setActiveForm('form4')}
			>
				Message details
			</button>
		</div>

		<div class="control-buttons">
			<button
				on:click={sendMessage}
				disabled={!apiAvailable || loading}
				class="button {!apiAvailable ? 'button--disabled' : ''}"
			>
				Write Content
			</button>
			<button on:click={runTest} class="button" disabled={!apiAvailable || loading}>
				(Demo for beta)
			</button>
			<button on:click={copy} class="button" disabled={loading || messages.length === 0}>
				Copy Content
			</button>
			<button on:click={clear} class="button" disabled={loading || messages.length === 0}>
				Reset All
			</button>
		</div>
	</div>

	{#each messages as { role, content, complete }, i}
		{#if role === 'progress'}
			<div class="message progress {complete ? 'completed' : ''}">
				<p>{@html content}</p>
			</div>
		{:else if role === 'assistant'}
			<div class="message {role}">
				<p>{@html content}</p>
			</div>
		{/if}
	{/each}
</main>

<hr />

<p>Here is the grab bag of input fields...</p>

<footer>
	{#if messages.length > maxMessages}
		<p>You reached the maximum amount of messages, you can either copy or reset</p>
		<button class="button" on:click={copy}>Copy Content</button>
		<button class="button" on:click={clear}>Reset All</button>
	{:else}
		<!-- Form container with conditional display based on active form -->
		<div class="form-container">
			<!-- Form 1 -->
			{#if activeForm === 'form1'}
				<form on:submit|preventDefault>
					{#each formSections_Research as section, sectionIndex}
						<h1>{section.title}</h1>
						{#each section.subsections as subsection, subsectionIndex}
							<h2>{subsection.title}</h2>

							{#each subsection.questions as question, questionIndex}
								{@const globalIndex = getQuestionIndex(question)}

								<p>{question}</p>
								<textarea
									placeholder="Type here (Question {globalIndex + 1})"
									bind:value={form2_input_arr[globalIndex]}
									on:keydown={handleKeyDown}
								></textarea>
							{/each}
						{/each}
					{/each}
				</form>
			{/if}

			<!-- Form 2 -->
			{#if activeForm === 'form2'}
				<form on:submit|preventDefault>
					<!-- Render form sections using the structured data -->
					{#each formSections_Target as section, sectionIndex}
						<h1>{section.title}</h1>
						{#each section.subsections as subsection, subsectionIndex}
							<h2>{subsection.title}</h2>

							<!-- Special handling for 'Content Requirements' to add an h3 for 'Precise Purpose' -->
							{#if subsection.title === 'Content Requirements'}
								<h3>Precise Purpose</h3>
							{/if}

							{#each subsection.questions as question, questionIndex}
								<!-- Calculate the global index for this question -->
								{@const globalIndex = getQuestionIndex(question)}

								<!-- Add special h3 headers for specific subsections -->
								{#if subsection.title === 'Supporting Evidence' && questionIndex === 0}
									<h3>Supporting Evidence</h3>
								{:else if subsection.title === 'Logical Structure' && questionIndex === 0}
									<h3>Logical Structure</h3>
								{/if}

								<p>{question}</p>
								<textarea
									placeholder="Type here (Question {globalIndex + 1})"
									bind:value={input_arr[globalIndex]}
									on:keydown={handleKeyDown}
								></textarea>
							{/each}
						{/each}
					{/each}
				</form>
			{/if}

			<!-- Form 3 -->
			{#if activeForm === 'form3'}
				<form on:submit|preventDefault>
					{#each formSections_Message as section, sectionIndex}
						<h1>{section.title}</h1>
						{#each section.subsections as subsection, subsectionIndex}
							<h2>{subsection.title}</h2>

							{#each subsection.questions as question, questionIndex}
								{@const globalIndex = getQuestionIndex(question)}

								<p>{question}</p>
								<textarea
									placeholder="Type here (Question {globalIndex + 1})"
									bind:value={form3_input_arr[globalIndex]}
									on:keydown={handleKeyDown}
								></textarea>
							{/each}
						{/each}
					{/each}
				</form>
			{/if}

			<!-- Form 4 -->
			{#if activeForm === 'form4'}
				<form on:submit|preventDefault>
					{#each formSections_MessageDetails as section, sectionIndex}
						<h1>{section.title}</h1>
						{#each section.subsections as subsection, subsectionIndex}
							<h2>{subsection.title}</h2>

							{#each subsection.questions as question, questionIndex}
								{@const globalIndex = getQuestionIndex(question)}

								<p>{question}</p>
								<textarea
									placeholder="Type here (Question {globalIndex + 1})"
									bind:value={form4_input_arr[globalIndex]}
									on:keydown={handleKeyDown}
								></textarea>
							{/each}
						{/each}
					{/each}
				</form>
			{/if}
		</div>
	{/if}
</footer>

<style>
	main {
		display: flex;
		flex-direction: column;
		gap: 1rem;
		width: 100%;
		max-width: 100%;
	}

	.header-section {
		border-bottom: 1px solid var(--text-subtle);
	}

	.header-section h1 {
		margin-top: 0;
		margin-bottom: 1rem;
		font-size: 2rem;
	}

	.header-section .notes {
		font-size: 0.7rem;
	}

	.warning-box {
		margin: 0.5rem 0;
		padding: 0.75rem 1.25rem;
		background-color: #fff3cd;
		border: 1px solid #ffeeba;
		border-radius: 0.25rem;
	}

	.warning {
		color: #856404;
		margin: 0.5rem 0;
	}

	.warning.error {
		color: #721c24;
		background-color: #f8d7da;
		border-color: #f5c6cb;
	}

	.top-buttons {
		display: flex;
		gap: 1rem;
		margin-top: 1.5rem;
		align-items: center;
	}

	.button--disabled,
	button[disabled] {
		opacity: 0.5;
		cursor: not-allowed;
		pointer-events: none; /* Prevents hover effects */
		background-color: #cccccc !important; /* Override other background colors */
		color: #666666 !important; /* Darker text */
		border: 1px solid #999999;
	}

	form {
		display: flex;
		flex-direction: column;
		width: 100%;
		max-width: 100%;
	}

	textarea {
		width: 100%;
		height: 100px;
		border: solid 1px var(--text);
		border-radius: 10px;
		padding: 10px;
		font-size: var(--font-size);
		box-sizing: border-box;
		font-family: var(--font-body);
		max-width: 100%;
	}

	.control-buttons {
		display: flex;
		gap: 1rem;
		align-items: center;
		margin: 0.5rem 0;
		flex-wrap: wrap;
	}

	/* Common button styles across all buttons */
	button {
		background-color: var(--brand);
		color: var(--bg);
		border: none;
		border-radius: 10px;
		padding: 10px;
		font-size: var(--font-size);
		font-family: var(--font-body);
		cursor: pointer;
		display: flex;
		align-self: flex-end;
		font-weight: bold;
		transition: background-color 0.2s ease;
	}

	button:hover {
		background-color: var(--brand-subtle);
	}

	button:active {
		background-color: var(--brand);
	}

	.message {
		display: flex;
		border-radius: 10px;
		border: solid 1px var(--text);
		justify-content: flex-start;
		white-space: pre-wrap;
		max-width: 100%;
	}

	.message p {
		margin: 0;
	}

	.user {
		flex-direction: row-reverse;
		justify-content: flex-end;
		margin-left: auto;
	}

	.assistant {
		border-color: var(--brand);
		flex-direction: row;
		justify-content: flex-start;
		margin-right: auto;
	}

	:global(.progress) {
		border-color: var(--text-subtle);
		background-color: var(--bg-subtle);
		color: var(--text);
		width: 100%;
		margin: 0.5rem 0;
		padding: 0.5rem 1rem;
		text-align: left;
		font-family: monospace;
		/* Add the loading animation while in progress */
		background-image: linear-gradient(90deg, var(--bg) 0%, var(--bg-subtle) 50%, var(--bg) 100%);
		background-size: 200% 100%;
		animation: loading 3s linear infinite;
	}

	:global(.progress ul) {
		margin: 0.5rem 0;
		padding-left: 1.5rem;
		list-style-type: square;
	}

	:global(.progress li) {
		margin: 0.25rem 0;
		position: relative;
	}

	:global(.progress li.current) {
		font-weight: bold;
	}

	:global(.progress li.pending) {
		font-style: italic;
	}

	:global(.progress strong) {
		display: block;
		margin-bottom: 0.5rem;
		font-size: 1.5em;
		font-weight: bold;
		text-align: center;
	}

	/* Only apply completed style when the 'Done' text is present */
	.progress.completed {
		animation: none;
		background-image: none;
		background-color: #c0ffc0; /* Light green background */
		border-color: #c3e6cb;
		color: #155724;
	}

	.loading {
		/* message loading bg animated */
		background-image: linear-gradient(90deg, var(--bg) 0%, var(--bg-subtle) 50%, var(--bg) 100%);
		background-size: 200% 100%;
		animation: loading 3s linear infinite;
	}

	@keyframes loading {
		0% {
			background-position: 100% 0;
		}
		100% {
			background-position: -100% 0;
		}
	}

	.message p {
		padding: 10px;
		border-radius: 10px;
	}
</style>
