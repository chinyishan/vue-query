<template>
  <TodoForm></TodoForm>
  <button @click="toogleComponent">
    {{ showComponent ? 'Umount' : 'Mount' }}
  </button>
  <TodoList v-if="showComponent"></TodoList>
  <VueQueryDevtools />
</template>

<script setup>
import { ref } from 'vue';
import TodoForm from './components/TodoForm.vue';
import TodoList from './components/TodoList.vue';
import { VueQueryDevtools } from '@tanstack/vue-query-devtools';

/**
 * 當 Mount 時會重新再打API取資料
 */
const showComponent = ref(true);

const toogleComponent = () => {
  showComponent.value = !showComponent.value;
};
</script>
