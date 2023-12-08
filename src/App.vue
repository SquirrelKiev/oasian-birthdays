<script setup lang="ts">
import { ref } from "vue";

const birthdayListIO = ref("");

const usernameInput = ref("@");
const birthdayInput = ref("2023-01-01");

const errText = ref("");

let birthdays: Birthday[] = [];

function onAddBirthdayButton() {
  const birthday = new Date(birthdayInput.value + "T12:00");

  birthdays.push({
    username: usernameInput.value,
    day: birthday.getUTCDate(),
    month: birthday.getUTCMonth(),
  });

  usernameInput.value = "@";
  birthdayInput.value = "2023-01-01";

  sortBirthdays();
  updateIO();
}

function onIOUpdatedByUser() {
  try {
    birthdays = parseBirthdays(birthdayListIO.value);
    sortBirthdays();
    updateIO();
  } catch (err) {
    let message = "Unknown Error";
    if (err instanceof Error) message = err.message;

    errText.value = message;

    return;
  }

  errText.value = "";
}

function sortBirthdays() {
  birthdays.sort((a, b) => {
    if (a.month !== b.month) {
      return a.month - b.month;
    }

    return a.day - b.day;
  });
}

function updateIO() {
  let birthdaysStr = "";

  birthdays.forEach((element, index) => {
    const timestamp = Math.floor(
      getFutureUnixTimestamp(element.day, element.month) / 1000
    ); // unneccessary floor but im being safe here

    birthdaysStr += `${element.username} - <t:${timestamp}:d> (<t:${timestamp}:R>)`;
    
    if (index < birthdays.length - 1) {
      birthdaysStr += '\n';
    }
  });

  birthdayListIO.value = birthdaysStr;
}

function getFutureUnixTimestamp(day: number, month: number): number {
  const currentDate = new Date();
  const currentYear = currentDate.getUTCFullYear();

  let yearForFutureDate = currentYear;
  if (
    month < currentDate.getUTCMonth() ||
    (month === currentDate.getUTCMonth() && day <= currentDate.getUTCDate())
  ) {
    yearForFutureDate++;
  }

  const futureDate = Date.UTC(yearForFutureDate, month, day, 12, 0, 0);

  return futureDate;
}

function parseBirthdays(input: string): Birthday[] {
  const lines = input.split("\n");

  return lines.map((line, index) => {
    const parts = line.split(/ - (?=<t:\d+:d>)/);

    if (parts.length !== 2) {
      throw new Error(`Invalid line format at line ${index + 1}`);
    }

    const username = parts[0];
    const timestampPart = parts[1];

    const timestampMatch = timestampPart.match(/<t:(\d+):d>/);
    if (!timestampMatch) {
      throw new Error(`Invalid timestamp format at line ${index + 1}`);
    }
    const timestamp = parseInt(timestampMatch[1], 10);

    const date = new Date(timestamp * 1000);

    const day = date.getUTCDate();
    const month = date.getUTCMonth();

    return { username, day, month };
  });
}

interface Birthday {
  username: string;
  day: number;
  month: number;
}
</script>

<template>
  <h1>Oasian birthday list maker thing</h1>
  <p>Tool for generating ever-updating birthday lists for Discord.</p>
  <textarea
    rows="10"
    v-model="birthdayListIO"
    @input="onIOUpdatedByUser"
    placeholder="The actual useable Discord text will be in here. You can also paste an existing list in and it'll be added, sorted, fixed etc"
  ></textarea>
  <p>{{ errText }}</p>

  <hr />
  <h3>Add a birthday</h3>
  <label for="username">Username</label>
  <input
    type="text"
    placeholder="Username"
    id="username-input"
    v-model="usernameInput"
  />
  <label for="birthday-input">Birthday (Year is ignored)</label>
  <input type="date" v-model="birthdayInput" />
  <button @click="onAddBirthdayButton">Add Birthday</button>
</template>

<style scoped></style>
