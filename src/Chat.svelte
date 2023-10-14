<script>
  import Login from './Login.svelte';
  import ChatMessage from './ChatMessage.svelte';
  import { onMount } from 'svelte';
  import { username, user } from './user';
  import debounce from 'lodash.debounce';
  import SimpleCrypto from 'simple-crypto-js'; // Import SimpleCrypto library

  import GUN from 'gun';
  const db = GUN();

  let newMessage;
  let messages = [];

  let scrollBottom;
  let lastScrollTop;
  let canAutoScroll = true;
  let unreadMessages = false;

  // Create an instance of SimpleCrypto with your secret key
  const secretKey = 'sahita'; // Replace with your actual secret key
  const simpleCrypto = new SimpleCrypto(secretKey);

  function autoScroll() {
    setTimeout(() => scrollBottom?.scrollIntoView({ behavior: 'auto' }), 50);
    unreadMessages = false;
  }

  function watchScroll(e) {
    canAutoScroll = (e.target.scrollTop || Infinity) > lastScrollTop;
    lastScrollTop = e.target.scrollTop;
  }

  $: debouncedWatchScroll = debounce(watchScroll, 1000);

  onMount(() => {
    var match = {
      '.': {
        '>': new Date(+new Date() - 1 * 1000 * 60 * 60 * 3).toISOString(),
      },
      '-': 1,
    };

    db.get('chat')
      .map(match)
      .once(async (data, id) => {
        if (data) {
          const key = '#foo';

          var message = {
            who: await db.user(data).get('alias'),
            what: simpleCrypto.decrypt(data.what), // Decrypt the message
            when: GUN.state.is(data, 'what'),
          };

          if (message.what) {
            console.log('Decrypted message:', message.what); // Add this line to check the decrypted message
            messages = [...messages.slice(-100), message].sort((a, b) => a.when - b.when);
            if (canAutoScroll) {
              autoScroll();
            } else {
              unreadMessages = true;
            }
          }
        }
      });
  });

  async function sendMessage() {
    const encryptedMessage = simpleCrypto.encrypt(newMessage); // Encrypt the message
    console.log('Encrypted message:', encryptedMessage); // Add this line to check the encrypted message
    const message = user.get('all').set({ what: encryptedMessage });
    const index = new Date().toISOString();
    db.get('chat').get(index).put(message);
    newMessage = '';
    canAutoScroll = true;
    autoScroll();
  }
</script>


<div class="container">
  {#if $username}
    <main on:scroll={debouncedWatchScroll}>
      {#each messages as message (message.when)}
        <ChatMessage {message} sender={$username} />
      {/each}

      <div class="dummy" bind:this={scrollBottom} />
    </main>

    <form on:submit|preventDefault={sendMessage}>
      <input type="text" placeholder="Type a message..." bind:value={newMessage} maxlength="100" />

      <button type="submit" disabled={!newMessage}>Send</button>
    </form>

    {#if !canAutoScroll}
    <div class="scroll-button">
      <button on:click={autoScroll} class:red={unreadMessages}>
        {#if unreadMessages}
          💬
        {/if}

        👇
      </button>
    </div>
   {/if}
  {:else}
    <main>
      <Login />
    </main>
  {/if}
</div>
