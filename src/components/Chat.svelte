<!-- src/components/Chat.svelte -->
<script>
  import { GoogleGenAI } from "@google/genai";
  import { onMount, tick } from "svelte";

  // --- Props ---
  // Use bind:messages in App.svelte to allow two-way binding
  export let messages = [];

  // --- State Variables (Local to Chat component) ---
  // let messages = []; // REMOVED - Now managed by App.svelte via prop
  let userInput = "";
  let isLoading = false;
  let chatContainer;
  let error = null;
  let initError = null;
  let ai;

  // --- Model Selection State ---
  const LOCAL_STORAGE_KEY_MODELS = "userAddedGeminiModels";
  const baseModels = [
    "gemini-1.5-flash-latest",
    "gemini-1.5-pro-latest",
    "gemini-1.0-pro",
  ];
  let availableModels = [...baseModels];
  let selectedModel = "";
  let showAddModelInput = false; //custom model input popup
  let newModelName = "";
  let addModelError = null;

  // --- API Setup ---
  const GEMINI_API_KEY = import.meta.env.VITE_GEMINI_API_KEY;

  // --- Lifecycle ---
  onMount(() => {
    // Load user models
    loadUserModels();
    if (availableModels.length > 0) {
      // Maybe load last used model from localStorage too? For now, default.
      selectedModel = availableModels[0];
    } else {
      initError = "No models available.";
    }

    // Initialize API (keep this logic)
    if (!GEMINI_API_KEY) {
      initError = "Config Error: VITE_GEMINI_API_KEY missing.";
      console.error(initError);
      return;
    }
    if (initError) return;

    try {
      ai = new GoogleGenAI({ apiKey: GEMINI_API_KEY });
      // Initial message is now handled by App.svelte when creating/loading a chat
      // We don't need to add it here anymore.
      // messages = [ ... ]; // REMOVED
      scrollToBottom(); // Scroll to bottom on initial load/chat switch
    } catch (e) {
      initError = `Error initializing GoogleGenAI: ${e.message}`;
      console.error(initError, e);
    }
  });

  // --- Functions ---

  // loadUserModels, saveUserModels, handleAddModelClick, etc. remain the same
  function loadUserModels() {
    try {
      const storedModelsJson = localStorage.getItem(LOCAL_STORAGE_KEY_MODELS);
      let userModels = [];
      if (storedModelsJson) {
        userModels = JSON.parse(storedModelsJson);
        if (!Array.isArray(userModels)) {
          // Basic validation
          console.warn(
            "Invalid data found in localStorage for models, resetting."
          );
          userModels = [];
          localStorage.removeItem(LOCAL_STORAGE_KEY_MODELS);
        }
      }
      // Combine base models and user models, ensuring uniqueness using a Set
      const combined = new Set([...baseModels, ...userModels]);
      availableModels = Array.from(combined);
    } catch (e) {
      console.error(
        "Failed to load or parse user models from localStorage:",
        e
      );
      // Fallback to just base models if loading fails
      availableModels = [...baseModels];
      localStorage.removeItem(LOCAL_STORAGE_KEY_MODELS); // Clear potentially corrupted data
      initError = "Warning: Could not load custom models from storage."; // Inform user non-critically
    }
  }

  // Save only the user-added models back to localStorage
  function saveUserModels() {
    try {
      // Filter out the base models to only store user-added ones
      const userModelsToSave = availableModels.filter(
        (m) => !baseModels.includes(m)
      );
      localStorage.setItem(
        LOCAL_STORAGE_KEY_MODELS,
        JSON.stringify(userModelsToSave)
      );
    } catch (e) {
      console.error("Failed to save user models to localStorage:", e);
      addModelError = "Could not save model list."; // Show error in the add model UI
    }
  }

  // Show the input field for adding a new model
  function handleAddModelClick() {
      showAddModelInput = true;
      newModelName = ''; // Clear previous input
      addModelError = null; // Clear previous error
  }

  // Hide the input field for adding a new model
  function handleCancelAddModel() {
      showAddModelInput = false;
      newModelName = '';
      addModelError = null;
  }

  // Validate and save the newly entered model name
  function handleSaveNewModel() {
      const trimmedName = newModelName.trim();
      addModelError = null; // Clear previous error

      if (!trimmedName) {
          addModelError = "Model name cannot be empty.";
          return;
      }
      if (availableModels.includes(trimmedName)) {
          addModelError = `Model "${trimmedName}" already exists.`;
          return;
      }

      // Add to the reactive list (Svelte updates the UI)
      availableModels = [...availableModels, trimmedName];

      // Persist the updated list of user models to localStorage
      saveUserModels();

      // Automatically select the newly added model
      selectedModel = trimmedName;

      // Hide the input UI
      showAddModelInput = false;
      newModelName = '';
  }

  async function scrollToBottom() {
    // Wait for DOM update AFTER messages prop potentially changes
    await tick();
    if (chatContainer) {
      chatContainer.scrollTop = chatContainer.scrollHeight;
    }
  }

  async function handleSubmit() {
    if (!userInput.trim() || isLoading || !ai || !selectedModel) return;

    const userMessageContent = userInput.trim();

    // *** IMPORTANT: Modify the messages array directly (bound prop) ***
    messages = [...messages, { role: "user", content: userMessageContent }];
    // No need to dispatch event, App.svelte's reactive block will catch the change.

    userInput = "";
    isLoading = true;
    error = null;
    scrollToBottom(); // Scroll after adding user message

    // Add placeholder for AI response
    messages = [...messages, { role: "model", content: "" }];
    const currentModelMessageIndex = messages.length - 1;

    try {
      const contents = [
        { role: "user", parts: [{ text: userMessageContent }] },
      ];

      const responseStream = await ai.models.generateContentStream({
        model: selectedModel,
        contents: contents,
      });

      let accumulatedText = "";
      for await (const chunk of responseStream) {
        const chunkText = chunk.text;
        if (chunkText) {
          accumulatedText += chunkText;
          // *** IMPORTANT: Modify the messages array directly ***
          messages[currentModelMessageIndex].content = accumulatedText;
          messages = messages; // Trigger Svelte reactivity for the array modification
          scrollToBottom();
        }
      }
      // Final update just in case
      messages[currentModelMessageIndex].content = accumulatedText;
      messages = messages; // Trigger reactivity
    } catch (e) {
      console.error(`Error calling Gemini API with model ${selectedModel}:`, e);
      error = `API Error (${selectedModel}): ${e.message || "Unknown error"}`;
      // Remove the placeholder message on error
      // *** IMPORTANT: Modify the messages array directly ***
      messages.pop();
      messages = messages; // Trigger reactivity
    } finally {
      isLoading = false;
      scrollToBottom();
    }
  }

  // --- Reactivity for Scrolling ---
  // Scroll when messages array changes (e.g., when loading a new chat)
  $: if (messages) {
    scrollToBottom();
  }
</script>

<!-- Component Root Element -->
<div class="flex flex-col h-full bg-base-200 min-w-0">
  <!-- Header -->
  <div class="navbar bg-base-300 shadow-md flex-shrink-0">
    <a class="btn btn-ghost normal-case text-xl">Gemini Chat</a>
    <div class="ml-auto text-sm opacity-70 hidden sm:block">
      Using: {selectedModel || "N/A"}
    </div>
  </div>

  <!-- Initialization Error Display -->
  {#if initError}
    <div class="alert alert-error shadow-lg rounded-none m-4">
      <div>
        <svg
          xmlns="http://www.w3.org/2000/svg"
          class="stroke-current flex-shrink-0 h-6 w-6"
          fill="none"
          viewBox="0 0 24 24"
          ><path
            stroke-linecap="round"
            stroke-linejoin="round"
            stroke-width="2"
            d="M10 14l2-2m0 0l2-2m-2 2l-2 2m2-2l2 2m7-2a9 9 0 11-18 0 9 9 0 0118 0z"
          /></svg
        >
        <span>{initError}</span>
      </div>
    </div>
  {/if}

  <!-- Chat Area -->
  <div
    class="flex-grow p-4 overflow-y-auto space-y-4"
    bind:this={chatContainer}
    aria-live="polite"
  >
    <!-- ... (message rendering loop remains the same) ... -->
    {#each messages as message, i (i)}
      {#if message.role === "user"}
        <div class="chat chat-end">
          <div class="chat-bubble chat-bubble-info overflow-hidden table-fixed">
            <p class="whitespace-pre-wrap break-words max-w-full text-white">{message.content}</p>
          </div>
        </div>
      {:else if message.role === "model"}
        <div class="chat chat-start">
          <div class="chat-image avatar">
            <div
              class="w-10 rounded-full bg-base-content/10 text-xl flex items-center justify-center ring ring-primary ring-offset-base-100 ring-offset-1"
            >
              ✨
            </div>
          </div>
          <div class="chat-bubble chat-bubble-neutral overflow-hidden table-fixed">
            <div class="prose prose-sm max-w-none text-white-content">
              {#if message.content}
              <pre class="whitespace-pre-wrap font-sans break-words max-w-full">{message.content}</pre>
              {:else if isLoading && i === messages.length - 1}
                <span class="loading loading-dots loading-sm"></span>
              {/if}
            </div>
          </div>
        </div>
      {/if}
    {/each}

    {#if error && !isLoading}
      <div class="chat chat-start">
        <div class="chat-bubble chat-bubble-error">
          {error}
        </div>
      </div>
    {/if}
  </div>

  <!-- Input Area -->
  <div class="p-4 bg-base-300 shadow-inner mt-auto flex-shrink-0">
    <!-- Model Selection & Add Area -->
    <div class="mb-2">
      <label for="model-select" class="label pb-1 pt-0">
        <span class="label-text">Select Model:</span>
      </label>
      <div class="flex items-center space-x-2">
        <select
          id="model-select"
          class="select select-bordered select-sm w-full max-w-xs"
          bind:value={selectedModel}
          disabled={isLoading || !!initError || showAddModelInput}
          aria-label="Select AI Model"
        >
          {#each availableModels as modelName (modelName)}
            <option value={modelName}>{modelName}</option>
          {/each}
        </select>
        {#if !showAddModelInput}
          <button
            class="btn btn-sm btn-square btn-ghost"
            on:click={handleAddModelClick}
            title="Add custom model"
            aria-label="Add custom model"
            disabled={isLoading || !!initError}
          >
            <!-- Plus Icon -->
            <svg
              xmlns="http://www.w3.org/2000/svg"
              fill="none"
              viewBox="0 0 24 24"
              stroke-width="1.5"
              stroke="currentColor"
              class="w-5 h-5"
            >
              <path
                stroke-linecap="round"
                stroke-linejoin="round"
                d="M12 4.5v15m7.5-7.5h-15"
              />
            </svg>
          </button>
        {/if}
      </div>
    </div>

    <!-- Add Model Input Area (Conditional) -->
    {#if showAddModelInput}
      <div
        class="mb-2 p-3 border border-base-content/20 rounded-md bg-base-100"
      >
        <label for="new-model-input" class="label pb-1 pt-0">
          <span class="label-text text-xs">Enter Custom Model Name:</span>
        </label>
        <div class="flex items-center space-x-2">
          <input
            type="text"
            id="new-model-input"
            placeholder="e.g., gemini-experimental"
            class="input input-bordered input-sm flex-grow"
            bind:value={newModelName}
            aria-label="New model name input"
            on:keydown={(e) => e.key === "Enter" && handleSaveNewModel()}
          />
          <button
            class="btn btn-sm btn-success btn-square"
            title="Save Model"
            aria-label="Save new model"
            on:click={handleSaveNewModel}
          >
            <!-- Check Icon -->
            <svg
              xmlns="http://www.w3.org/2000/svg"
              fill="none"
              viewBox="0 0 24 24"
              stroke-width="1.5"
              stroke="currentColor"
              class="w-5 h-5"
            >
              <path
                stroke-linecap="round"
                stroke-linejoin="round"
                d="M4.5 12.75l6 6 9-13.5"
              />
            </svg>
          </button>
          <button
            class="btn btn-sm btn-ghost btn-square"
            title="Cancel"
            aria-label="Cancel adding model"
            on:click={handleCancelAddModel}
          >
            <!-- X Icon -->
            <svg
              xmlns="http://www.w3.org/2000/svg"
              fill="none"
              viewBox="0 0 24 24"
              stroke-width="1.5"
              stroke="currentColor"
              class="w-5 h-5"
            >
              <path
                stroke-linecap="round"
                stroke-linejoin="round"
                d="M6 18L18 6M6 6l12 12"
              />
            </svg>
          </button>
        </div>
        {#if addModelError}
          <p class="text-error text-xs mt-1">{addModelError}</p>
        {/if}
      </div>
    {/if}

    <!-- Message Input Form -->
    <form on:submit|preventDefault={handleSubmit} class="flex space-x-2">
      <input
        type="text"
        placeholder={isLoading
          ? "Waiting for response..."
          : "Type your message..."}
        class="input input-bordered w-full flex-grow"
        bind:value={userInput}
        disabled={isLoading || !!initError || showAddModelInput}
        aria-label="Chat message input"
      />
      <button
        type="submit"
        class="btn btn-primary"
        disabled={isLoading ||
          !userInput.trim() ||
          !!initError ||
          showAddModelInput}
        aria-label="Send message"
      >
        {#if isLoading}
          <span class="loading loading-spinner loading-sm" aria-hidden="true"
          ></span>
          <span class="sr-only">Sending...</span>
        {:else}
          Send
        {/if}
      </button>
    </form>
  </div>
</div>

<style>
  /* ... (styles remain the same) ... */
  .prose pre {
    background-color: transparent;
    padding: 0;
    margin: 0;
    color: inherit;
    font-family: inherit;
    overflow-x: auto;
    font-size: 0.9em;
  }
  .prose p {
    margin: 0;
  }
  .flex-grow {
    min-height: 0;
  }
</style>
