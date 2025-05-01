<!-- src/App.svelte -->
<script>
  import { onMount } from 'svelte';
  import Chat from './components/Chat.svelte'; // Adjust path if needed
  import ChatHistory from './components/ChatHistory.svelte'; // Adjust path if needed

  const CHAT_HISTORY_KEY = 'geminiChatHistory';
  const ACTIVE_CHAT_ID_KEY = 'geminiActiveChatId';

  let savedChats = []; // Array of { id: number, createdAt: number, messages: [] }
  let activeChatId = null;
  let activeChatMessages = []; // The messages array for the currently active chat
  let isInitialized = false; // Prevent rendering Chat until state is loaded

  // --- Initialization ---
  onMount(() => {
    loadHistory();
    isInitialized = true;
  });

  // --- Helper Functions ---
  function createNewChatState() {
      const newId = Date.now(); // Use timestamp as a simple unique ID
      return {
          id: newId,
          createdAt: newId,
          messages: [
              // Start with the initial greeting from the model
              { role: 'model', content: 'Hello! How can I help you today?' }
          ]
      };
  }

  function findChatIndex(id) {
      return savedChats.findIndex(chat => chat.id === id);
  }

  // --- State Loading/Saving ---
  function loadHistory() {
      try {
          const storedHistory = localStorage.getItem(CHAT_HISTORY_KEY);
          const storedActiveId = localStorage.getItem(ACTIVE_CHAT_ID_KEY);

          if (storedHistory) {
              savedChats = JSON.parse(storedHistory);
              // Basic validation
              if (!Array.isArray(savedChats)) savedChats = [];
          } else {
              savedChats = [];
          }

          let foundActive = false;
          if (storedActiveId) {
              const parsedId = parseInt(storedActiveId, 10); // Or keep as string if IDs are strings
              const activeIndex = findChatIndex(parsedId);
              if (activeIndex !== -1) {
                  activeChatId = parsedId;
                  activeChatMessages = savedChats[activeIndex].messages;
                  foundActive = true;
              }
          }

          // If no history, no active ID, or active ID not found, start a new chat
          if (!foundActive) {
              startNewChatSession(); // This will create the first chat if needed
          }

      } catch (e) {
          console.error("Failed to load chat history from localStorage:", e);
          savedChats = [];
          startNewChatSession(); // Start fresh if loading failed
          // Optionally clear potentially corrupted localStorage
          localStorage.removeItem(CHAT_HISTORY_KEY);
          localStorage.removeItem(ACTIVE_CHAT_ID_KEY);
      }
  }

  function saveHistory() {
      if (!isInitialized) return; // Don't save before loading is complete
      try {
          // Ensure the currently active messages are up-to-date in savedChats
          const activeIndex = findChatIndex(activeChatId);
          if (activeIndex !== -1) {
              savedChats[activeIndex].messages = activeChatMessages;
              // Trigger reactivity for ChatHistory if needed (though direct mutation often works)
              savedChats = savedChats;
          }

          localStorage.setItem(CHAT_HISTORY_KEY, JSON.stringify(savedChats));
          localStorage.setItem(ACTIVE_CHAT_ID_KEY, activeChatId.toString());
      } catch (e) {
          console.error("Failed to save chat history to localStorage:", e);
      }
  }

  // --- Event Handlers from ChatHistory ---
  function handleNewChat() {
      // Save current state before switching
      saveHistory();
      // Start new session
      startNewChatSession();
      // Save again to persist the new empty chat and active ID
      saveHistory();
  }

  function startNewChatSession() {
      const newChat = createNewChatState();
      // Add the new chat to the beginning of the history list
      savedChats = [newChat, ...savedChats];
      activeChatId = newChat.id;
      activeChatMessages = newChat.messages;
  }

  function handleLoadChat(event) {
      const idToLoad = event.detail.id;
      if (idToLoad === activeChatId) return; // Already active

      // Save current state before switching
      saveHistory();

      const chatIndex = findChatIndex(idToLoad);
      if (chatIndex !== -1) {
          activeChatId = idToLoad;
          activeChatMessages = savedChats[chatIndex].messages;
          // No need to call saveHistory() here, as loading doesn't change the stored data itself
          // Just update the active ID in localStorage for next page load
          localStorage.setItem(ACTIVE_CHAT_ID_KEY, activeChatId.toString());
      } else {
          console.error(`Chat with ID ${idToLoad} not found.`);
          // Optionally handle this error, e.g., start a new chat
      }
  }

  // --- Reactivity: Save whenever active messages change ---
  // $: is Svelte's reactive declaration. This block runs whenever its dependencies change.
  $: if (isInitialized && activeChatMessages) {
      // This automatically saves the history when messages in the active chat are updated by Chat.svelte
      saveHistory();
  }

</script>

<main class="flex h-screen w-screen overflow-hidden">
  {#if isInitialized}
    <!-- Sidebar -->
    <ChatHistory
      {savedChats}
      {activeChatId}
      on:newChat={handleNewChat}
      on:loadChat={handleLoadChat}
    />

    <!-- Main Chat Window -->
    <div class="flex-grow h-full min-w-0">
      <!-- Pass the messages for the *active* chat down to the Chat component -->
      <!-- Use bind:messages to allow Chat.svelte to directly modify the array -->
      <Chat bind:messages={activeChatMessages} />
    </div>
  {:else}
     <!-- Optional: Loading indicator while history loads -->
     <div class="flex items-center justify-center w-full h-full">
         <span class="loading loading-lg loading-spinner text-primary"></span>
     </div>
  {/if}
</main>

<style>
  /* Ensure no default body margin interferes */
  :global(body) {
    margin: 0;
  }
  /* Ensure main takes full viewport */
  main {
      height: 100vh;
      width: 100vw;
  }
</style>