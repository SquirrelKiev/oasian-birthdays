<script setup lang="ts">
import { ref } from "vue";

const birthdayListIO = ref("");

const usernameInput = ref("@");
const birthdayInput = ref("2023-01-01");

const errText = ref("");

const shouldSortRelative = ref(true);

let birthdays: Birthday[] = [];
let header: string = "";
let footer: string = "";

function onAddBirthdayButton() {
  const birthday = new Date(birthdayInput.value + "T12:00");

  birthdays.push({
    username: usernameInput.value,
    day: birthday.getUTCDate(),
    month: birthday.getUTCMonth(),
  });

  usernameInput.value = "@";
  birthdayInput.value = "2024-01-01";

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
  if (shouldSortRelative.value) {
    sortBirthdaysRelative();
  } else {
    sortBirthdaysAbsolute();
  }
}

function sortBirthdaysAbsolute() {
  birthdays.sort((a, b) => {
    if (a.month !== b.month) {
      return a.month - b.month;
    }
    return a.day - b.day;
  });
}

function sortBirthdaysRelative() {
  const currentDate = new Date();
  const currentYear = currentDate.getUTCFullYear();

  birthdays.sort((a, b) => {
    const dateA = new Date(Date.UTC(currentYear, a.month, a.day));
    const dateB = new Date(Date.UTC(currentYear, b.month, b.day));

    if (dateA < currentDate) dateA.setUTCFullYear(currentYear + 1);
    if (dateB < currentDate) dateB.setUTCFullYear(currentYear + 1);

    return dateA.getTime() - dateB.getTime();
  });
}

function updateIO() {
  let birthdaysStr = header ? `${header}` : "";

  if (birthdays.length > 0) {
    if (header) birthdaysStr += "\n";
    birthdays.forEach((element, index) => {
      const timestamp = Math.floor(
        getFutureUnixTimestamp(element.day, element.month) / 1000
      ); // unneccessary floor but im being safe here
      birthdaysStr += `${element.username} - <t:${timestamp}:d> (<t:${timestamp}:R>)`;
      if (index < birthdays.length - 1) {
        birthdaysStr += "\n";
      }
    });
  }

  if (footer) birthdaysStr += `\n${footer}`;

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
  let birthdayListStart = -1;
  let birthdayListEnd = -1;

  // find the start and end of the birthday list
  for (let i = 0; i < lines.length; i++) {
    if (isBirthdayLine(lines[i])) {
      if (birthdayListStart === -1) birthdayListStart = i;
      birthdayListEnd = i;
    }
  }

  if (birthdayListStart === -1) {
    header = input;
    footer = "";
    return [];
  }

  // get the header and footer
  header = lines.slice(0, birthdayListStart).join("\n");
  footer = lines.slice(birthdayListEnd + 1).join("\n");

  // get and parse the birthday list
  return lines
    .slice(birthdayListStart, birthdayListEnd + 1)
    .map(parseBirthdayLine);
}

function isBirthdayLine(line: string): boolean {
  return /@.* - <t:\d+:d>/.test(line);
}

function parseBirthdayLine(line: string, index: number): Birthday {
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
  <input
    type="checkbox"
    id="relative-sort-toggle"
    v-model="shouldSortRelative"
    @change="onIOUpdatedByUser"
  />
  <label for="relative-sort-toggle"
    >Should the list be sorted relative to the current date? (birthdays that
    have passed go to the end of the list)</label
  >
  <hr />
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
