<template>
  <div v-if="initialSyncComplete">
    <div class="controls">
      <div>
        <button @mousedown="addMessages" @mouseup="stopAction">
          Add Messages
        </button>
        <button @mousedown="removeMessages" @mouseup="stopAction">
          Remove Messages
        </button>
        <em>(hold buttons to repeat)</em>
      </div>
      <div style="justify-content: end">
        <span v-if="user !== 'anon'">Logged in as {{ user }}</span>
        <button @mousedown="toggleLogin">
          {{ user === "anon" ? "Login" : "Logout" }}
        </button>
      </div>
    </div>

    <div class="controls">
      <div>
        From:
        <select @change="(e) => (filterUser = e.target.value)" style="flex: 1">
          <option value="">Sender</option>
          <option v-for="usr in users" :key="usr.id" :value="usr.id">
            {{ usr.name }}
          </option>
        </select>
      </div>
      <div>
        By:
        <select
          @change="(e) => (filterMedium = e.target.value)"
          style="flex: 1"
        >
          <option value="">Medium</option>
          <option v-for="med in mediums" :key="med.id" :value="med.id">
            {{ med.name }}
          </option>
        </select>
      </div>
      <div>
        Contains:
        <input
          type="text"
          placeholder="message"
          @input="(e) => (filterText = e.target.value)"
          style="flex: 1"
        />
      </div>
      <div>
        After:
        <input
          type="date"
          @change="(e) => (filterDate = e.target.value)"
          style="flex: 1"
        />
      </div>
    </div>

    <div class="controls">
      <em>
        <template v-if="!hasFilters">
          Showing all {{ filteredMessages.length }} messages
        </template>
        <template v-else>
          Showing {{ filteredMessages.length }} of
          {{ allMessages.length }} messages. Try opening
          <a href="/" target="_blank">another tab</a> to see them all!
        </template>
      </em>
    </div>

    <div v-if="filteredMessages.length === 0">
      <h3><em>No posts found 😢</em></h3>
    </div>
    <div v-else>
      <table class="messages">
        <thead>
          <tr>
            <th>Sender</th>
            <th>Medium</th>
            <th>Message</th>
            <th>Sent</th>
            <th>Edit</th>
          </tr>
        </thead>
        <tbody>
          <tr v-for="message in filteredMessages" :key="message.id">
            <td>{{ message.sender?.name }}</td>
            <td>{{ message.medium?.name }}</td>
            <td>{{ message.body }}</td>
            <td>{{ formatDate(message.timestamp) }}</td>
            <td
              @mousedown="
                (e) =>
                  editMessage(e, message.id, message.senderID, message.body)
              "
            >
              ✏️
            </td>
          </tr>
        </tbody>
      </table>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, computed, watchEffect } from "vue";
import Cookies from "js-cookie";
import { useQuery } from "@rocicorp/zero/vue";
import { escapeLike, Zero } from "@rocicorp/zero";
import { Schema } from "./schema";
import { randomMessage } from "./test-data";
import { randInt } from "./rand";
import { formatDate } from "./date";

const props = defineProps<{ z: Zero<Schema> }>();

const users = useQuery(() => props.z.query.user);
const mediums = useQuery(() => props.z.query.medium);

const filterUser = ref("");
const filterMedium = ref("");
const filterText = ref("");
const filterDate = ref("");

const allMessages = useQuery(() => props.z.query.message);

const filteredMessages = useQuery(() => {
  let filtered = props.z.query.message
    .related("medium", (medium) => medium.one())
    .related("sender", (sender) => sender.one())
    .orderBy("timestamp", "desc");

  if (filterUser.value) {
    filtered = filtered.where("senderID", filterUser.value);
  }

  if (filterMedium.value) {
    filtered = filtered.where("mediumID", filterMedium.value);
  }

  if (filterText.value) {
    filtered = filtered.where(
      "body",
      "LIKE",
      `%${escapeLike(filterText.value)}%`,
    );
  }

  if (filterDate.value) {
    filtered = filtered.where(
      "timestamp",
      ">=",
      new Date(filterDate.value).getTime(),
    );
  }
  return filtered;
});

const hasFilters = computed(
  () =>
    filterUser.value ||
    filterMedium.value ||
    filterText.value ||
    filterDate.value,
);

const action = ref<"add" | "remove" | undefined>(undefined);

watchEffect(() => {
  if (action.value !== undefined) {
    const interval = setInterval(() => {
      if (!handleAction()) {
        clearInterval(interval);
        action.value = undefined;
      }
    }, 1000 / 60);
  }
});

const handleAction = () => {
  if (action.value === undefined) {
    return false;
  }
  if (action.value === "add") {
    props.z.mutate.message.insert(randomMessage(users.value, mediums.value));
    return true;
  } else {
    const messages = allMessages.value;
    if (messages.length === 0) {
      return false;
    }
    const index = randInt(messages.length);
    props.z.mutate.message.delete({ id: messages[index].id });
    return true;
  }
};

const addMessages = () => {
  action.value = "add";
};

const removeMessages = (e: MouseEvent) => {
  if (props.z.userID === "anon" && !e.shiftKey) {
    alert("You must be logged in to delete. Hold the shift key to try anyway.");
    return;
  }
  action.value = "remove";
};

const stopAction = () => {
  action.value = undefined;
};

const editMessage = (
  e: MouseEvent,
  id: string,
  senderID: string,
  prev: string,
) => {
  if (senderID !== props.z.userID && !e.shiftKey) {
    alert(
      "You aren't logged in as the sender of this message. Editing won't be permitted. Hold the shift key to try anyway.",
    );
    return;
  }
  const body = prompt("Edit message", prev);
  props.z.mutate.message.update({
    id,
    body: body ?? prev,
  });
};

const toggleLogin = async () => {
  if (props.z.userID === "anon") {
    await fetch("/api/login");
  } else {
    Cookies.remove("jwt");
  }
  location.reload();
};

const initialSyncComplete = computed(
  () => users.value.length && mediums.value.length,
);

const user = computed(
  () => users.value.find((u) => u.id === props.z.userID)?.name ?? "anon",
);
</script>
