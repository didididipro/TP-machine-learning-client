<script setup lang="ts">
import logo from './assets/logo.webp'
</script>



<template>
	<header class="flex justify-center">
		<img width="120px" :src="logo">
	</header>
	<router-view/>
</template>