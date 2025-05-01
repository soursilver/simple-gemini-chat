<!-- src/components/ChatHistory.svelte -->
<script>
    import { createEventDispatcher } from 'svelte';
  
    export let savedChats = []; // { id: number, createdAt: number, messages: [], title?: string }[]
    export let activeChatId = null;
  
    const dispatch = createEventDispatcher();
  
    function handleNewChat() {
      dispatch('newChat');
    }
  
    function handleLoadChat(id) {
      if (id !== activeChatId) {
        dispatch('loadChat', { id });
      }
    }
  
    // Helper to format date
    function formatChatDate(timestamp) {
        const date = new Date(timestamp);
        return date.toLocaleDateString('en-US', { month: 'short', day: 'numeric' }).toUpperCase(); // e.g., JUL 4
    }
  
    // Helper to format time
    function formatChatTime(timestamp) {
        const date = new Date(timestamp);
        return date.toLocaleTimeString('en-US', { hour: 'numeric', minute: '2-digit', hour12: true }); // e.g., 10:35 AM
    }
  </script>
  
  <div class="w-64 bg-base-300 h-full flex flex-col border-r border-base-content/10 flex-shrink-0">
    <!-- New Chat Button -->
    <div class="p-2 border-b border-base-content/10">
      <button
        class="btn btn-primary btn-sm w-full"
        on:click={handleNewChat}
      >
        <!-- Plus Icon -->
         <svg xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke-width="1.5" stroke="currentColor" class="w-4 h-4 mr-1">
           <path stroke-linecap="round" stroke-linejoin="round" d="M12 4.5v15m7.5-7.5h-15" />
         </svg>
        New Chat
      </button>
    </div>
  
    <!-- Chat History List -->
    <div class="flex-grow overflow-y-auto p-2 space-y-1">
      {#if savedChats.length === 0}
          <p class="text-xs text-center opacity-50 p-4">No chat history yet.</p>
      {/if}
      {#each savedChats as chat (chat.id)}
        <button
          class="w-full text-left p-2 rounded-md hover:bg-base-100 focus:outline-none focus:ring-1 focus:ring-primary {chat.id === activeChatId ? 'bg-primary/20 font-semibold' : ''}"
          on:click={() => handleLoadChat(chat.id)}
          aria-current={chat.id === activeChatId ? 'page' : undefined}
        >
          <div class="text-sm font-medium truncate">{formatChatDate(chat.createdAt)}</div>
          <div class="text-xs opacity-70 truncate">{formatChatTime(chat.createdAt)}</div>
           <!-- Optional: Show first user message as preview -->
           <!-- {#if chat.messages.find(m => m.role === 'user')}
              <p class="text-xs opacity-50 truncate mt-1">
                  {chat.messages.find(m => m.role === 'user').content}
              </p>
           {/if} -->
        </button>
      {/each}
    </div>
  
     <!-- Optional Footer -->
     <!-- <div class="p-2 border-t border-base-content/10 text-center text-xs opacity-50">
         Chat History
     </div> -->
  </div>