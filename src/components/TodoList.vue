<template>
  <h2 v-if="isTodosLoading">Loading...</h2>
  <ul>
    <li v-for="item in todos" :key="item.id">
      <span v-if="editingTodoId === item.id">
        <input type="text" v-model="editingTodoTitle" />
        <button @click="updateTodo(item.id)" :disabled="isUpdatingTodo">
          {{ isUpdatingTodo ? 'Updating...' : 'Update' }}
        </button>
      </span>
      <span v-else
        >{{ item.title }}
        <button @mouseover="prefetchTodo(item.id)" @click="startEditTodo(item)">
          Edit
        </button>
        <button @click="deleteTodo(item.id)" :disabled="isDeletingTodo">
          {{ isDeletingTodo ? 'Deleting' : 'Delete' }}
        </button>
      </span>
    </li>
  </ul>
  <h3 v-if="isSelecteTodoLoading">selected loading...</h3>

  {{ selectedTodoData }}
</template>
<script setup>
import { ref, computed } from 'vue';
import { useQuery, useMutation, useQueryClient } from '@tanstack/vue-query';
import apiClient from '../apiClient';

const editingTodoId = ref(null);
const editingTodoTitle = ref('');
const selectedTodoId = ref(null);

const queryClient = useQueryClient();

/**獲取todos列表 */
const {
  data: todosData,
  isLoading: isTodosLoading,
  refetch,
} = useQuery({
  queryKey: ['todos'],
  queryFn: async () => {
    const { data } = await apiClient.get('/todos');
    return data;
  },
  // staleTime: Infinity, // 資料多久不新鮮，Infinity: 只發送一次
  // refetchInterval: 5000, // 每幾毫秒發一次API
  // gcTime: 5000, // 把快取放掉，快取要保留多久
  // retry: 1, // API request error，預設會重新發三次
  // retryDelay: 1, // API request error，每己毫秒重發
  // enabled: false, // 禁止發送API
});

const todos = computed(() => todosData.value || []);

/**選擇編輯todo */
const { data: selectedTodoData, isLoading: isSelecteTodoLoading } = useQuery({
  queryKey: ['todo', selectedTodoId],
  queryFn: async () => {
    const { data } = await apiClient.get(`/todos/${selectedTodoId.value}`);
    return data;
  },
  // !! : null或是空值就為false
  enabled: computed(() => !!selectedTodoId.value),
  staleTime: 5000,
});

/** prefetch 預先存取資料，使用 mouseover 揣測使用者行為 */
const prefetchTodo = async (id) => {
  await queryClient.prefetchQuery({
    queryKey: ['todo', id],
    queryFn: async () => {
      const { data } = await apiClient.get(`/todos/${id}`);
      return data;
    },
    staleTime: 5000,
  });
};

/**編輯todos Mutation */
const { mutate: updateTodoMutate, isPending: isUpdatingTodo } = useMutation({
  mutationFn: async (updateTodo) => {
    console.log('mutationFn', updateTodo);

    // onMutate 測試錯誤用
    if (updateTodo.title.trim() === 'update-fail') {
      await new Promise((resolve) => setTimeout(resolve, 1000));
      return Promise.reject('update-fail');
    }

    const { data } = await apiClient.put(`/todos/${updateTodo.id}`, updateTodo);
    return data;
  },
  // 使用 onMutate 做樂觀更新，前端畫面先做資料更新在打API，做樂觀更新也需要rollback
  onMutate: async (updateTodo) => {
    console.log('onMutate', updateTodo);
    cancelEdit();
    // 進行樂觀更新之前，避免舊資料覆蓋先針對 queryKey 全部處理掉，使用 cancelQueries
    await queryClient.cancelQueries(['todos']);

    // getQueryData 取得原始資料，用於錯誤時可以取得
    const previousTodos = queryClient.getQueryData(['todos']);

    // setQueryData 重新寫資料
    queryClient.setQueryData(['todos'], (oldTodos) => {
      return oldTodos.map((todo) =>
        todo.id === updateTodo.id ? { ...todo, ...updateTodo } : todo
      );
    });

    // 回傳原始
    return { previousTodos };
  },
  // 錯誤，重寫原始資料
  onError: (err, updateTodo, context) => {
    console.log(context, 'onError context');
    queryClient.setQueryData(['todos'], context.previousTodos);
  },
  // 做重新查詢
  // onSettled: () => {
  // // 使缓存中的每个查询都无效
  //   queryClient.invalidateQueries(['todos']);
  // },
  onSuccess: () => {
    refetch();
    cancelEdit();
  },
});

/**刪除todo Mutation */
const { mutate: deleteTodoMutate, isPending: isDeletingTodo } = useMutation({
  mutationFn: async (todoId) => {
    const { data } = await apiClient.delete(`/todos/${todoId}`);
    return data;
  },
  onSuccess: () => {
    refetch();
  },
});

/**開始編輯 */
const startEditTodo = (todo) => {
  selectedTodoId.value = todo.id;
  editingTodoId.value = todo.id;
  editingTodoTitle.value = todo.title;
};

/**做更新資料 */
const updateTodo = (todoId) => {
  updateTodoMutate({
    id: todoId,
    title: editingTodoTitle.value,
  });
};

/**取消編輯 */
const cancelEdit = () => {
  selectedTodoId.value = null;
  editingTodoId.value = null;
  editingTodoTitle.value = '';
};

/**刪除 */
const deleteTodo = (todoId) => {
  deleteTodoMutate(todoId);
};
</script>
