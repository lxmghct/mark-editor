<template>
  <div>
    <div class="home">
      <h1>Home</h1>
      <button @click="handleClick">click</button>
    </div>
  </div>
</template>

<script>
export default {
  methods: {
    handleClick() {
      console.log('click')
    }
  }
}
</script>

<style scoped>
</style>
