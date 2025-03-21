<script setup>
import { RouterLink, RouterView } from 'vue-router'
</script>

<template>
  <nav class="navbar bg-body-tertiary mb-3">
    <div class="container-fluid">
      <a class="navbar-brand" href="#">connectedAlex</a>
    </div>
  </nav>

  <!--  <header>
    <nav>
      <RouterLink to="/">Home</RouterLink>
      <RouterLink to="/about">About</RouterLink>
    </nav>
  </header>-->

  <RouterView />
</template>

<style scoped></style>
