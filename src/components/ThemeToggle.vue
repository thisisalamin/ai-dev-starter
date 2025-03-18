<template>
    <Button variant="outline" size="icon" @click="toggleTheme">
      <Sun v-if="isDark" class="h-[1.2rem] w-[1.2rem] rotate-0 scale-100 transition-all" />
      <Moon v-else class="h-[1.2rem] w-[1.2rem] rotate-0 scale-100 transition-all" />
      <span class="sr-only">Toggle theme</span>
    </Button>
  </template>
  
  <script setup lang="ts">
  import { ref, onMounted } from 'vue';
  import { Button } from '@/components/ui/button';
  import { Moon, Sun } from 'lucide-vue-next';
  
  const isDark = ref(false);
  
  onMounted(() => {
    // Check if user prefers dark mode
    if (window.matchMedia && window.matchMedia('(prefers-color-scheme: dark)').matches) {
      isDark.value = true;
      document.documentElement.classList.add('dark');
    }
    
    // Check for saved theme preference
    const savedTheme = localStorage.getItem('theme');
    if (savedTheme) {
      isDark.value = savedTheme === 'dark';
      if (isDark.value) {
        document.documentElement.classList.add('dark');
      } else {
        document.documentElement.classList.remove('dark');
      }
    }
  });
  
  const toggleTheme = () => {
    isDark.value = !isDark.value;
    
    if (isDark.value) {
      document.documentElement.classList.add('dark');
      localStorage.setItem('theme', 'dark');
    } else {
      document.documentElement.classList.remove('dark');
      localStorage.setItem('theme', 'light');
    }
  };
  </script>