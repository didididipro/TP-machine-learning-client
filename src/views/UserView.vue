<script setup lang="ts">
import { ref } from 'vue';
import logo from './assets/logo.webp'

</script>



<template>
	<header class="flex justify-center gap-4">
		<img width="100px" :src="logo">
	</header>
	<router-view/>

</template>