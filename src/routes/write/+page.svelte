<script lang="ts">
	import { botName } from '$lib/config'
	import { onMount, onDestroy } from 'svelte'

	// SVELTEKIT SUBSTITUTE: Moved shared types to avoid importing from Netlify Functions
	type ChatResponse = {
		response: string
		apiAvailable?: boolean
		stateToken?: string
		progressString?: string
		complete?: boolean
		information?: string
		jobId?: string
		status?: string
	}

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
	const FORM_DATA_STORAGE_KEY = 'email_writer_form_data'
	const JOB_STORAGE_KEY = 'email_writer_job_id'

	let messages: Message[] =
		typeof localStorage !== 'undefined' ? JSON.parse(localStorage.getItem(STORAGE_KEY) || '[]') : []

	// Array for form
	let loading = false
	let apiAvailable = true // Default to true, will be updated after first API call
	const maxMessages = 20

	// Job tracking variables
	let currentJobId: string | null = null
	// SVELTEKIT SUBSTITUTE: Fixed timer types to avoid TypeScript errors
	let pollingInterval: ReturnType<typeof setInterval> | null = null
	let jobTimeout: ReturnType<typeof setTimeout> | null = null
	const POLLING_INTERVAL_MS = 2000 // Poll every 2 seconds
	const JOB_TIMEOUT_MS = 300000 // 5 minutes timeout

	// Organizing the form questions into sections and subsections
	const formSections_Target: FieldSection[] = [
		{
			title: 'Researching a person',
			subsections: [
				{
					title: 'To research a person, fill out the following fields.',
					questions: ["Person's Name", 'Current Role', 'Organization/Affiliation']
				}
			]
		},
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
					questions: ['What might incentivize them to act', 'Their likely concerns or interests']
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
					]
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
			title: 'Message Details',
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
					questions: ['Urgency of the request', 'Specific deadlines']
				},
				{
					title: 'Supplementary Information',
					questions: ['Attachments needed', 'Links to additional resources']
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

	const paragraphText_MessageDetails: string[] = []
	formSections_MessageDetails.forEach((section) => {
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
					questions: ['Specific outcome desired', 'Concrete action requested']
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

	// Helper functions for form management
	function getCurrentInputArray(): string[] {
		switch (activeForm) {
			case 'form1':
				return form1_input_arr
			case 'form2':
				return form2_input_arr
			case 'form3':
				return form3_input_arr
			case 'form4':
				return form4_input_arr
			case 'form5':
				return form2_input_arr.concat(form3_input_arr, form4_input_arr)
			default:
				return form2_input_arr
		}
	}

	function getCurrentQuestionArray(): string[] {
		switch (activeForm) {
			case 'form1':
				return paragraphText_Research
			case 'form2':
				return paragraphText_Target
			case 'form3':
				return paragraphText_Message
			case 'form4':
				return paragraphText_MessageDetails
			case 'form5':
				let allText = paragraphText_Target.concat(
					paragraphText_Message,
					paragraphText_MessageDetails
				)
				return allText
			default:
				return paragraphText_Target
		}
	}

	function clear_arr(arr: string[]) {
		for (var i in arr) {
			arr[i] = ''
		}
	}

	function clear() {
		// Stop any active polling
		stopJobPolling()

		messages = []
		localStorage.setItem(STORAGE_KEY, JSON.stringify(messages))

		// Clear all form arrays
		clear_arr(form1_input_arr)
		clear_arr(form2_input_arr)
		clear_arr(form3_input_arr)
		clear_arr(form4_input_arr)

		// Clear form data and job ID from localStorage
		localStorage.removeItem(FORM_DATA_STORAGE_KEY)
		localStorage.removeItem(JOB_STORAGE_KEY)

		// Clear current job ID
		currentJobId = null
	}

	function copy() {
		const role = (message: Message) => (message.role === 'user' ? 'You' : 'Writer')
		const text = messages.map((message) => `${role(message)}:\n${message.content}`).join('\n\n')
		navigator.clipboard.writeText(text)
		window.alert(
			"Copied to clipboard\n\nDisclaimer: Check all text content carefully before sending anything!\n\nYou are the one sending the email and expressing an opinion. You've simply had some assistance in writing it."
		)
	}

	function runTest() {
		// Clear any existing chat
		clear()

		// Get current form arrays
		const currentInputArray = getCurrentInputArray()
		const currentQuestionArray = getCurrentQuestionArray()

		// Clear current input fields
		clear_arr(currentInputArray)

		// Provide test data based on active form
		switch (activeForm) {
			case 'form1':
				// Test data for research form
				if (currentInputArray.length > 0) {
					currentInputArray[0] =
						'Local city council member or school board representative who handles education policy and child safety issues. Location: California, Los Angeles.'
				}
				break

			case 'form2':
				// Test data for target/personal context form
				const roleAuthorityIndex = currentQuestionArray.findIndex(
					(q) => q === 'Understanding their role and potential authority'
				)
				if (roleAuthorityIndex >= 0) {
					currentInputArray[roleAuthorityIndex] =
						'You are writing for a local government official with decision-making authority over education and safety policies.'
				}
				break

			case 'form3':
				// Test data for message form
				const objectiveIndex = currentQuestionArray.findIndex(
					(q) => q === 'Concrete action requested'
				)
				const outcomeIndex = currentQuestionArray.findIndex((q) => q === 'Specific outcome desired')

				if (objectiveIndex >= 0) {
					currentInputArray[objectiveIndex] =
						'Official takes action to ensure AI safety education and policies are implemented in local institutions.'
				}
				if (outcomeIndex >= 0) {
					currentInputArray[outcomeIndex] =
						'Local community becomes informed about AI risks and appropriate safety measures are put in place.'
				}
				break

			case 'form4':
				// Test data for message details form
				const urgencyIndex = currentQuestionArray.findIndex((q) => q === 'Urgency of the request')
				const toneIndex = currentQuestionArray.findIndex(
					(q) => q === 'Balancing professionalism and approachability'
				)

				if (urgencyIndex >= 0) {
					currentInputArray[urgencyIndex] =
						'High urgency due to rapidly advancing AI development timeline.'
				}
				if (toneIndex >= 0) {
					currentInputArray[toneIndex] =
						'Professional but accessible tone that conveys seriousness without being alarmist.'
				}
				break
		}

		sendMessage()
	}

	// New job polling functions
	function startJobPolling(jobId: string) {
		// Clear any existing polling
		stopJobPolling()

		currentJobId = jobId
		localStorage.setItem(JOB_STORAGE_KEY, jobId)

		// SVELTEKIT SUBSTITUTE: Fixed timer types
		// Start polling
		pollingInterval = setInterval(async () => {
			await checkJobStatus(jobId)
		}, POLLING_INTERVAL_MS)

		// Set timeout to prevent infinite polling
		jobTimeout = setTimeout(() => {
			stopJobPolling()
			console.warn('Job polling timed out')
			messages = [
				...messages,
				{
					content: 'The job is taking longer than expected. Please try again or contact support.',
					role: 'assistant'
				}
			]
			loading = false
		}, JOB_TIMEOUT_MS)

		console.log(`Started polling for job ${jobId}`)
	}

	function stopJobPolling() {
		if (pollingInterval) {
			clearInterval(pollingInterval)
			pollingInterval = null
		}

		if (jobTimeout) {
			clearTimeout(jobTimeout)
			jobTimeout = null
		}
	}

	async function checkJobStatus(jobId: string) {
		try {
			// SVELTEKIT SUBSTITUTE: Updated API endpoint to use Netlify Functions
			const response = await fetch(`/.netlify/functions/status?jobId=${jobId}`)
			const data = await response.json()

			if (data.error) {
				console.error('Error checking job status:', data.error)
				return
			}

			// Update progress message
			if (data.progressString) {
				const progressIndex = messages.findIndex((m) => m.role === 'progress')
				if (progressIndex >= 0) {
					messages[progressIndex].content = data.progressString
					messages[progressIndex].complete = data.complete
				} else {
					messages = [
						...messages,
						{
							content: data.progressString,
							role: 'progress',
							complete: data.complete
						}
					]
				}
			}

			// Update email content
			if (data.response) {
				const assistantIndex = messages.findIndex((m) => m.role === 'assistant')
				if (assistantIndex >= 0) {
					messages[assistantIndex].content = data.response
				} else {
					messages = [
						...messages,
						{
							content: data.response,
							role: 'assistant'
						}
					]
				}
			}

			// Handle auto-fill information
			if (data.information) {
				updateFormFieldsFromInformation(data.information)
			}

			// Save messages to localStorage
			localStorage.setItem(STORAGE_KEY, JSON.stringify(messages))
			saveFormData()

			// Check if job is complete or failed
			if (data.status === 'completed' || data.status === 'failed' || data.complete) {
				stopJobPolling()
				loading = false

				if (data.status === 'failed') {
					messages = [
						...messages,
						{
							content: `Job failed: ${data.error || 'Unknown error occurred'}`,
							role: 'assistant'
						}
					]
				}

				console.log(`Job ${jobId} ${data.status}`)
			}
		} catch (error) {
			console.error('Error checking job status:', error)
		}
	}

	function updateFormFieldsFromInformation(information: string) {
		const currentInputArray = getCurrentInputArray()
		const currentQuestionArray = getCurrentQuestionArray()

		// Parse the information string to extract field values
		const lines = information.split('\n')
		let currentField = -1

		for (let line of lines) {
			// Look for field headers matching our current question array
			const fieldIndex = currentQuestionArray.findIndex((text) =>
				line.trim().startsWith(text + ':')
			)

			if (fieldIndex >= 0) {
				currentField = fieldIndex
			} else if (currentField >= 0 && line.trim()) {
				// Only update when there's actual content and the field is empty
				const lineContent = line.trim()
				if (
					lineContent &&
					(!currentInputArray[currentField] || currentInputArray[currentField] === '')
				) {
					currentInputArray[currentField] = lineContent
				}
			}
		}
	}

	async function sendMessage() {
		const currentInputArray = getCurrentInputArray()
		const currentQuestionArray = getCurrentQuestionArray()

		let input = ''
		switch (activeForm) {
			case 'form1':
				input = input + '[1]'
				break
			case 'form2':
				input = input + '[2]'
				break
			case 'form4':
				input = input + '[3]'
				break
			case 'form5':
				input = input + '[4]'
				break
		}

		for (let i = 0; i < currentQuestionArray.length; i++) {
			input =
				input + currentQuestionArray[i] + ':\n' + (currentInputArray[i] || 'undefined') + '\n\n'
		}

		messages = [...messages, { content: input, role: 'user' }]

		// Clear current form fields
		clear_arr(currentInputArray)
		loading = true
		console.log(input)

		try {
			// SVELTEKIT SUBSTITUTE: Updated API endpoint to use Netlify Functions
			// Create new job request
			const response = await fetch('/.netlify/functions/write', {
				method: 'POST',
				headers: {
					'Content-Type': 'application/json'
				},
				body: JSON.stringify([{ content: input, role: 'user' }])
			})

			const data = await response.json()

			if (!data.apiAvailable) {
				apiAvailable = false
				messages = [
					...messages,
					{
						content: data.response || 'API not available',
						role: 'assistant'
					}
				]
				loading = false
				return
			}

			// Add initial progress message
			if (data.progressString) {
				messages = [
					...messages,
					{
						content: data.progressString,
						role: 'progress',
						complete: data.complete || false
					}
				]
			}

			// Handle auto-fill information
			if (data.information) {
				updateFormFieldsFromInformation(data.information)
			}

			// Save messages and form data
			localStorage.setItem(STORAGE_KEY, JSON.stringify(messages))
			saveFormData()

			// Start polling if we got a job ID
			if (data.jobId) {
				startJobPolling(data.jobId)
			} else {
				// Job might be already complete
				if (data.response) {
					messages = [
						...messages,
						{
							content: data.response,
							role: 'assistant'
						}
					]
				}
				loading = false
			}
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
			loading = false
		}
	}

	// Form data persistence functions
	function saveFormData() {
		const formData = {
			form1_input_arr,
			form2_input_arr,
			form3_input_arr,
			form4_input_arr,
			activeForm
		}
		localStorage.setItem(FORM_DATA_STORAGE_KEY, JSON.stringify(formData))
	}

	function loadFormData() {
		const saved = localStorage.getItem(FORM_DATA_STORAGE_KEY)
		if (saved) {
			const formData = JSON.parse(saved)
			form1_input_arr = formData.form1_input_arr || new Array<string>(paragraphText_Research.length)
			form2_input_arr = formData.form2_input_arr || new Array<string>(paragraphText_Target.length)
			form3_input_arr = formData.form3_input_arr || new Array<string>(paragraphText_Message.length)
			form4_input_arr =
				formData.form4_input_arr || new Array<string>(paragraphText_MessageDetails.length)
			activeForm = formData.activeForm || 'form1'
		}
	}

	async function resumeJobIfExists() {
		const savedJobId = localStorage.getItem(JOB_STORAGE_KEY)
		if (savedJobId) {
			console.log(`Resuming job monitoring for ${savedJobId}`)
			// Check if job is still active
			try {
				// SVELTEKIT SUBSTITUTE: Updated API endpoint to use Netlify Functions
				const response = await fetch(`/.netlify/functions/status?jobId=${savedJobId}`)
				const data = await response.json()

				if (!data.error && (data.status === 'pending' || data.status === 'processing')) {
					// Resume polling for active job
					loading = true
					startJobPolling(savedJobId)
				} else if (data.status === 'completed') {
					// Job completed while away, update UI
					if (data.response) {
						const assistantIndex = messages.findIndex((m) => m.role === 'assistant')
						if (assistantIndex >= 0) {
							messages[assistantIndex].content = data.response
						} else {
							messages = [
								...messages,
								{
									content: data.response,
									role: 'assistant'
								}
							]
						}
					}
					localStorage.removeItem(JOB_STORAGE_KEY)
				} else {
					// Job failed or not found, clean up
					localStorage.removeItem(JOB_STORAGE_KEY)
				}
			} catch (error) {
				console.error('Error resuming job:', error)
				localStorage.removeItem(JOB_STORAGE_KEY)
			}
		}
	}

	onMount(async () => {
		loadFormData()

		// Check API availability on component mount
		try {
			// SVELTEKIT SUBSTITUTE: Updated API endpoint to use Netlify Functions
			const response = await fetch('/.netlify/functions/write')
			const data = await response.json()
			apiAvailable = !!data.apiAvailable
		} catch (error) {
			console.error('Error checking API availability:', error)
			apiAvailable = false
		}

		// Resume job monitoring if there's an active job
		await resumeJobIfExists()
	})

	onDestroy(() => {
		// Clean up polling when component is destroyed
		stopJobPolling()
	})

	function handleKeyDown(event: KeyboardEvent) {
		// Key handling for forms if needed
	}

	// FORM FUNCTIONS //
	let activeForm = 'form1' // Default active form

	function setActiveForm(formId: string) {
		saveFormData() // Save current state before switching
		activeForm = formId
		saveFormData() // Save current state after switching to save activeForm
		console.log(formId)
	}

	function writeMail() {
		setActiveForm('form5')
		sendMessage()
	}

	// Create separate arrays for each form's inputs
	let form1_input_arr = Array(paragraphText_Research.length).fill('')
	let form2_input_arr = Array(paragraphText_Target.length).fill('')
	let form3_input_arr = Array(paragraphText_Message.length).fill('')
	let form4_input_arr = Array(paragraphText_MessageDetails.length).fill('')

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
				disabled={!apiAvailable || loading || activeForm === 'form3'}
				class="button {!apiAvailable ? 'button--disabled' : ''}"
			>
				Autofill
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
			<button
				on:click={writeMail}
				disabled={!apiAvailable || loading}
				class="button {!apiAvailable ? 'button--disabled' : ''}"
			>
				Write Mail
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
			<!-- Form 1 - Finding a Target -->
			{#if activeForm === 'form1'}
				<form on:submit|preventDefault>
					{#each formSections_Research as section, sectionIndex}
						<h1>{section.title}</h1>
						{#each section.subsections as subsection, subsectionIndex}
							<h2>{subsection.title}</h2>

							{#each subsection.questions as question, questionIndex}
								{@const globalIndex = paragraphText_Research.findIndex((text) => text === question)}

								<p>{question}</p>
								<textarea
									placeholder="Type here (Question {globalIndex + 1})"
									bind:value={form1_input_arr[globalIndex]}
									on:keydown={handleKeyDown}
								></textarea>
							{/each}
						{/each}
					{/each}
				</form>
			{/if}

			<!-- Form 2 - Personal Context -->
			{#if activeForm === 'form2'}
				<form on:submit|preventDefault>
					{#each formSections_Target as section, sectionIndex}
						<h1>{section.title}</h1>
						{#each section.subsections as subsection, subsectionIndex}
							<h2>{subsection.title}</h2>

							{#if subsection.title === 'Content Requirements'}
								<h3>Precise Purpose</h3>
							{/if}

							{#each subsection.questions as question, questionIndex}
								{@const globalIndex = paragraphText_Target.findIndex((text) => text === question)}

								{#if subsection.title === 'Supporting Evidence' && questionIndex === 0}
									<h3>Supporting Evidence</h3>
								{:else if subsection.title === 'Logical Structure' && questionIndex === 0}
									<h3>Logical Structure</h3>
								{/if}

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

			<!-- Form 3 - The Message -->
			{#if activeForm === 'form3'}
				<form on:submit|preventDefault>
					{#each formSections_Message as section, sectionIndex}
						<h1>{section.title}</h1>
						{#each section.subsections as subsection, subsectionIndex}
							<h2>{subsection.title}</h2>

							{#each subsection.questions as question, questionIndex}
								{@const globalIndex = paragraphText_Message.findIndex((text) => text === question)}

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

			<!-- Form 4 - Message Details -->
			{#if activeForm === 'form4'}
				<form on:submit|preventDefault>
					{#each formSections_MessageDetails as section, sectionIndex}
						<h1>{section.title}</h1>
						{#each section.subsections as subsection, subsectionIndex}
							<h2>{subsection.title}</h2>

							{#each subsection.questions as question, questionIndex}
								{@const globalIndex = paragraphText_MessageDetails.findIndex(
									(text) => text === question
								)}

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
		pointer-events: none;
		background-color: #cccccc !important;
		color: #666666 !important;
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

	button.active {
		background-color: var(--brand-subtle);
		font-weight: bold;
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
		padding: 10px;
		border-radius: 10px;
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

	.progress.completed {
		animation: none;
		background-image: none;
		background-color: #c0ffc0;
		border-color: #c3e6cb;
		color: #155724;
	}

	.loading {
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
</style>
