<script setup>
import { computed, onMounted, reactive, ref, watch } from 'vue';

const props = defineProps({ initialItems: { type: Array, default: () => [] } });
const items = ref(props.initialItems);
const keyword = ref('');
const total = ref(0);
const seconds = ref(0);
const state = reactive({ status: 'pending' });
const { status } = state;

const sortedItems = computed(() =>
  items.value.sort((left, right) => left.name.localeCompare(right.name))
);

watch(keyword, async (value) => {
  const response = await fetch(`/api/items?keyword=${encodeURIComponent(value)}`);
  items.value = await response.json();
});

watch(items, (value) => {
  total.value = value.length;
});

onMounted(() => {
  setInterval(() => { seconds.value += 1; }, 1000);
});

function renameFirstItem () {
  if (items.value.length) items.value[0].name = 'Updated';
}

function finish () {
  state.status = 'done';
}
</script>

<template>
  <section>
    <h2>Anti-patterns: learning only</h2>
    <input v-model="keyword" placeholder="Search" />
    <p>{{ total }} / {{ status }} / {{ seconds }}</p>
    <button @click="renameFirstItem">Rename first item</button>
    <button @click="finish">Finish</button>
    <ul>
      <li v-for="(item, index) in sortedItems" :key="index">
        {{ item.name }}
        <input placeholder="Item note" />
      </li>
    </ul>
  </section>
</template>
