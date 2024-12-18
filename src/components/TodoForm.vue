<template>
  <form>
    <input type="text" v-model="title" />
    <button @click="createTodo()" :disabled="isCreatingTodo">
      {{ isCreatingTodo ? 'Creating...' : 'Create' }}
    </button>
  </form>
</template>
<script setup>
import { ref } from 'vue';
import { useMutation, useQueryClient } from '@tanstack/vue-query';
import apiClient from '../apiClient';

const queryClient = useQueryClient();
const title = ref('');

const { mutate: createTodoMutate, isPending: isCreatingTodo } = useMutation({
  mutationFn: async (newTodo) => {
    const { data } = await apiClient.post('/todos', newTodo);
    return data;
  },
  onSuccess: () => {
    queryClient.invalidateQueries({ queryKey: ['todos'] });
  },
});

const createTodo = () => {
  createTodoMutate({
    id: crypto.randomUUID(),
    title: title.value,
  });
};
</script>
