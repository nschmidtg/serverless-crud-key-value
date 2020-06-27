<template>
  <div>
    <p>
      {{ item.key }}: {{ item.value }}
    </p>
    <button @click="deleteItem($event)">Delete</button>
  </div>
</template>

<script>
export default {
  name: 'item',
  props: {
    item: {
      type: Object,
      default: null
    }
  },
  methods: {
    deleteItem ($event) {
      this.$emit('delete-item', this.item)
    }
  }
}
</script>
