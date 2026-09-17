---
date: "2026-09-16T00:00:00+09:00"
title: "Centralized Vuetify Snackbar"
tags: ["Vue.js", "Vuetify"]
---

Vuetify offers the `<v-snackbar>` component which is often used as follows in applications:

```html
<template>
  <v-btn
    prepend-icon="mdi-content-save"
    text="Save"
    @click="save()"
    :loading="saving"
    color="primary"
  />

  <v-snackbar :color="snackbar.color" v-model="snackbar.show">
    {{ snackbar.text }}
  </v-snackbar>
</template>

<script lang="ts" setup>
  const saving = ref(false);

  const snackbar = ref({
    color: "success",
    show: false,
    text: "",
  });

  async function updateFood() {
    saving.value = true;

    try {
      await $fetch(`/api/items`, {
        method: "POST",
        body: {
          // ...
        },
      });
      snackbar.value.show = true;
      snackbar.value.text = "Saved successfully";
      snackbar.value.color = "success";
    } catch (error) {
      console.error(error);
      snackbar.value.show = true;
      snackbar.value.text = "Save failed";
      snackbar.value.color = "error";
    } finally {
      saving.value = false;
    }
  }
</script>
```

Having to repeat this pattern in every component that uses a snackbar creates a lot of code duplication. To solve this problem, one can introduce a centralized snackbar controlled by a composable.

```html
<template>
  <v-app>
    <v-main>
      <!-- ... -->
    </v-main>
    <v-snackbar
      :text="snackbar.text"
      v-model="snackbar.show"
      :color="snackbar.color"
    />
  </v-app>
</template>
<script setup>
  const { snackbar } = useSnackbar();
</script>
```

The `useSnackbar` composable can be defined as follows:

```ts
// composables/useSnackBar.ts
export function useSnackbar() {
  const snackbar = useState("snackbar", () => ({
    show: false,
    text: "",
    color: "success" as "success" | "error",
  }));

  function notify(text: string, color: "success" | "error" = "success") {
    snackbar.value = { show: true, text, color };
  }

  return { snackbar, notify };
}
```

The snackbar's visibility can then be triggered from any component using the `notify` function, which greatly simplifies the example above:

```html
<template>
  <v-btn
    prepend-icon="mdi-content-save"
    text="Save"
    @click="save()"
    :loading="saving"
    color="primary"
  />
</template>

<script lang="ts" setup>
  const saving = ref(false);

  async function updateFood() {
    saving.value = true;

    try {
      await $fetch(`/api/items`, {
        method: "POST",
        body: {
          // ...
        },
      });
      notify("Saved successfully");
    } catch (error) {
      console.error(error);
      notify("Save failed", "error");
    } finally {
      saving.value = false;
    }
  }
</script>
```
