<script lang="ts">
  import cx from "classnames";
  import { fly, fade } from "svelte/transition";
  import NDK, { NDKNip07Signer, NDKEvent } from "@nostr-dev-kit/ndk";

  const now = () => Math.round(Date.now() / 1000);

  const changeDay = (d, x) => {
    const date = new Date(d);

    date.setDate(d.getDate() + x);

    return date;
  };

  const dateToSeconds = (date) => Math.round(date.valueOf() / 1000);

  const generateMonth = (date) => ({
    date,
    days: getDaysInMonth(date.getFullYear(), date.getMonth()),
  });

  const getTimezoneOffset = () => new Date().getTimezoneOffset() * 60000;

  const fromNostrTime = (seconds) =>
    new Date(seconds * 1000 - getTimezoneOffset())
      .toISOString()
      .slice(0, -5)
      .split("T");

  const toNostrTime = (date, time) =>
    dateToSeconds(new Date(`${date}T${time}`)).toString();

  function getDaysInMonth(year, month) {
    const days = [];
    const firstDay = new Date(year, month, 1).getDay();
    const daysInThisMonth = new Date(year, month + 1, 0).getDate();
    const daysInLastMonth = new Date(year, month, 0).getDate();
    const prevMonth = month == 0 ? 11 : month - 1;

    // Show the days before the start of this month (disabled) - always less than 7
    for (let i = daysInLastMonth - firstDay; i < daysInLastMonth; i++) {
      days.push(new Date(prevMonth === 11 ? year - 1 : year, prevMonth, i + 1));
    }

    // Show the days in this month (enabled) - always 28 - 31
    for (let i = 0; i < daysInThisMonth; i++) {
      days.push(new Date(year, month, i + 1));
    }

    // Show any days to fill up the last row (disabled) - always less than 7
    for (let i = 0; days.length % 7; i++) {
      days.push(new Date(month == 11 ? year + 1 : year, (month + 1) % 12, i + 1));
    }

    return days;
  }

  const getMeta = ({ tags }) => {
    const meta = {};

    for (const [k, v] of tags) {
      meta[k] = v;
    }

    return meta;
  };

  const eventIsInRange = (date, event) => {
    const meta = getMeta(event);
    const startOfDay = dateToSeconds(date.setHours(0, 0, 0, 0));
    const endOfDay = dateToSeconds(date.setHours(23, 59, 59, 59));

    return parseInt(meta.start) <= endOfDay && parseInt(meta.end) >= startOfDay;
  };

  const getDateEvents = (date) =>
    events
      .filter((e) => eventIsInRange(date, e))
      .sort((a, b) => (getMeta(a).start > getMeta(b).start ? 1 : -1));

  const pendingNdk = (async () => {
    const ndk = new NDK({
      explicitRelayUrls: [
        "wss://nos.lol",
        "wss://nostr-pub.wellorder.net",
        "wss://relay.nostr.band",
      ],
    });

    await ndk.connect();

    return ndk;
  })();

  let user;
  let draft;
  let key = Math.random();
  let events = [];
  let month = generateMonth(new Date());

  const prevMonth = () => {
    const y = month.date.getMonth() === 0 ? month.date.getFullYear() - 1 : month.date.getFullYear();
    const m = month.date.getMonth() === 0 ? 11 : month.date.getMonth() - 1;

    month = generateMonth(new Date(y, m, 1));
  };

  const nextMonth = () => {
    const y = month.date.getMonth() === 11 ? month.date.getFullYear() + 1 : month.date.getFullYear();
    const m = month.date.getMonth() === 11 ? 0 : month.date.getMonth() + 1;

    month = generateMonth(new Date(y, m, 1));
  };

  const login = async () => {
  try {
    const ndk = await pendingNdk;

    ndk.signer = new NDKNip07Signer();

    await ndk.signer.blockUntilReady();

    user = await ndk.signer.user();

    loadCalendar();
  } catch (error) {
    // Handle the error by displaying it as an alert
    alert(` ${error.message} , if you're IOS user, visit this URL "Cal.NostrNet.work"`);
  }
};


  const loadCalendar = async () => {
    const ndk = await pendingNdk;

    const filter = { kinds: [31923], limit: 1000 };

    const allEvents = Array.from(await ndk.fetchEvents(filter));

    const deletions = Array.from(
      await ndk.fetchEvents({
        kinds: [5],
        "#a": Array.from(allEvents).map((e) => e.tagReference()[1]),
      })
    );

    const deletedAddresses = new Set(deletions.map((e) => e.tagValue("a")));

    events = allEvents.filter((e) => !deletedAddresses.has(e.tagReference()[1]));
    key = Math.random();
  };

  const newEvent = () => {
    const [date, time] = fromNostrTime(now());

    draft = {
      id: crypto.randomUUID(),
      name: "My Event",
      content: "My Description",
      startDate: date,
      startTime: time,
      endDate: date,
      endTime: time,
    };
  };

  const publishEvent = async () => {
    const ndk = await pendingNdk;
    const event = new NDKEvent(ndk, {
      kind: 31923,
      content: draft.content,
      tags: [
        ["d", draft.id],
        ["name", draft.name],
        ["start", toNostrTime(draft.startDate, draft.startTime)],
        ["end", toNostrTime(draft.endDate, draft.endTime)],
      ],
    });

    await event.publish();

    events = events.filter((e) => getMeta(e).d !== draft.id).concat(event);
    key = Math.random();

    draft = null;
  };

  const editEvent = async (e) => {
    const meta = getMeta(e);
    const [startDate, startTime] = fromNostrTime(meta.start);
    const [endDate, endTime] = fromNostrTime(meta.end);

    draft = {
      event: e,
      id: meta.d,
      name: meta.name,
      content: e.content,
      startDate,
      startTime,
      endDate,
      endTime,
    };
  };

  const clearEvent = async () => {
    draft = null;
  };

  const deleteEvent = async () => {
    if (confirm("Are you sure you want to delete this event?")) {
      await draft.event.delete();

      events = events.filter((e) => getMeta(e).d !== draft.id);
      key = Math.random();

      draft = null;
    }
  };

  

  

  loadCalendar();
</script>

<div class="absolute inset-0 flex flex-col max-w-screen px-0.5 lg:max-w-2xl mx-auto bg-[#18181a]">
  <div class="p-4 flex justify-between grid grid-cols-3">
    <h1 class="text-xl font-semibold text-blue-500">Calendar+</h1>
    <div class="flex justify-center items-center gap-2">
      <span class="p-1 pt-3 text-gray-400 cursor-pointer text-xs" on:click={prevMonth}>◀</span>
      <span class="font-bold w-36 text-sm text-gray-400 text-center">
        {month.date.toLocaleString("default", { month: "long", year: "numeric" })}
      </span>
      <span class="p-1 pt-3 cursor-pointer text-gray-400 text-xs" on:click={nextMonth}>▶</span>
    </div>
    <div class="flex justify-end">
      {#if user}
        <button class="text-[#3b82f7] text-sm font-bold py-1" on:click={newEvent}>Add Event</button>
      {:else}
        <button class="text-[#3b82f7] text-sm font-bold py-1" on:click={login}>Log In</button>
      {/if}
    </div>
  </div>
  <div class="grid grid-cols-7 mx-2 card-default-t card-default-r card-default-l" style="border-color: #808080;">
    <div class="text-center text-gray-400 text-xs py-2">Sun</div>
    <div class="text-center text-gray-400 text-xs py-2">Mon</div>
    <div class="text-center text-gray-400 text-xs py-2">Tue</div>
    <div class="text-center text-gray-400 text-xs py-2">Wed</div>
    <div class="text-center text-gray-400 text-xs py-2">Thu</div>
    <div class="text-center text-gray-400 text-xs py-2">Fri</div>
    <div class="text-center text-gray-400 text-xs py-2">Sat</div>
    </div>
  
  <!-- Add a row for day names -->
  <div class="grid grid-cols-7 mx-2 card-default-t card-default-l" style="border-color: #808080;">

    
     {#key key}
    {#each month.days as date, i}
      <div class="aspect-square text-gray-500 text-xs relative">
        <div class="absolute inset-0  card-default-r  card-default-b" style="border-color: #808080;"> </div>
        <div class="p-1">
          <!-- Add a click event handler to each date cell -->
          <div
            class="cursor-pointer text-xs whitespace-nowrap"
            on:click={() => openAddEventForm(date)}
          >
            {date.getDate()}
          </div>
          <div class="flex flex-col h-16  gap-1">
            <div class="relative z-10 cursor-pointer p-0.5 whitespace-nowrap overflow-y-auto" style="max-height: auto;">
              {#each getDateEvents(date) as event}
                {@const meta = getMeta(event)}
                {@const isContinuation = eventIsInRange(changeDay(date, -1), event)}
                {@const isContinued = eventIsInRange(changeDay(date, 1), event)}
                {@const isOwn = event.pubkey === user?.hexpubkey()}
                <div
                  class={cx(
                    "cursor-pointer text-xs whitespace-nowrap",
                    {
                      "z-20 bg-black text-[#3b82f7] border text-xs border-solid border-blue-500 hidden": !isOwn,
                      " my-1 text-[#3b82f7] px-0.5 border border-[#3b82f7]": isOwn,
                      "-ml-1 border-l-0": isContinuation,
                      "rounded-s": !isContinuation,
                      "-mr-1 border-r-0": isContinued,
                      "rounded-e text-ellipsis overflow-hidden": !isContinued,
                      "text-white/0": isContinuation && date.getDay() !== 0,
                    }
                  )}
                  on:click={() => editEvent(event)}
                >
                  {meta.name}
                </div>
              {/each}
            </div>
          </div>
        </div>
      </div>
    {/each}
  {/key}
</div>

  <small class="p-2 text-gray-400 text-center">
    Calendar+ is powered by NIP-52, managed by 
    <a href="https://www.nostrnet.work/" class="text-blue-500" target="_blank" rel="noopener noreferrer">NostrNet</a>
  </small>
  {#if draft}
    {@const isEditable = (!draft.event && user) || draft.event?.pubkey === user?.hexpubkey()}
    <div class="fixed z-20 inset-0 flex items-center justify-center bg-black/50 p-4" on:click={clearEvent}>
    <div in:fly={{ y: 20 }} class="bg-[#252528] lg:max-w-2xl rounded-xl p-4 flex flex-col gap-2" on:click|stopPropagation>
        <h2 class="text-xl text-gray-300 font-bold">Event Details</h2>
        <div class="grid grid-cols-3 text-gray-300 font-semibold items-center gap-2">
          <label for="name">Name</label>
          <input disabled={!isEditable} name="name" type="text" class="padding bg-[#18181a] card-default col-span-2 rounded-xl border-none shadow-xl font-semibold text-sm pl-2 p-1" bind:value={draft.name} />
          <label for="description">Description</label>
          <input disabled={!isEditable} name="description" type="text" class="padding bg-[#18181a] card-default col-span-2 rounded-xl border-none shadow-xl font-semibold text-sm pl-2 p-1" bind:value={draft.content} />
          <label for="start">Start</label>
          <input disabled={!isEditable} name="startDate" type="date" class="padding bg-[#18181a] card-default col-span-2 sm:col-span-1 rounded-xl border-none shadow-xl font-semibold text-sm pl-2 p-1" bind:value={draft.startDate} />
          <div class="col-span-1 sm:hidden"></div>
          <input disabled={!isEditable} name="startTime" type="time" class="padding bg-[#18181a] card-default col-span-2 sm:col-span-1 rounded-xl border-none shadow-xl font-semibold text-sm pl-2 p-1" bind:value={draft.startTime} />
          <label for="end">End</label>
          <input disabled={!isEditable} name="endDate" type="date" class="padding bg-[#18181a] card-default col-span-2 sm:col-span-1 rounded-xl border-none shadow-xl font-semibold text-sm pl-2 p-1" bind:value={draft.endDate} />
          <div class="col-span-1 sm:hidden"></div>
          <input disabled={!isEditable} name="endTime" type="time" class="padding bg-[#18181a] card-default col-span-2 sm:col-span-1 rounded-xl border-none shadow-xl font-semibold text-sm pl-2 p-1" bind:value={draft.endTime} />
        </div>
        {#if isEditable}
          <div class="flex justify-end gap-2">
            {#if draft.event}
              <button class="text-red-300  text-md font-bold py-2" on:click={deleteEvent}>Delete</button>
            {/if}
            <button class="text-[#3b82f7] text-md font-bold py-2 " on:click={publishEvent}>Save Event</button>
          </div>
        {/if}
      </div>
    </div>
  {/if}
</div>
