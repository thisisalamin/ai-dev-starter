<template>
  <div class="min-h-screen bg-background flex flex-col">
    <header class="border-b">
      <div class="container mx-auto px-4 py-4 flex items-center justify-between">
        <h1 class="text-2xl font-bold">AI Developer Starter</h1>
        <ThemeToggle />
      </div>
    </header>
    
    <main class="flex-1 container mx-auto px-4 py-6">
      <div class="grid grid-cols-1 md:grid-cols-3 gap-4">
        <div class="md:col-span-2">
          <ChatInterface />
        </div>
        <div class="md:col-span-1">
          <ProjectInfo />
        </div>
      </div>
    </main>
    
    <footer class="border-t py-4">
      <div class="container mx-auto px-4 text-center text-sm text-muted-foreground">
        &copy; 2023 AI Developer Starter. All rights reserved.
      </div>
    </footer>
  </div>
</template>

<script setup lang="ts">
import ChatInterface from './components/ChatInterface.vue';
import ProjectInfo from './components/ProjectInfo.vue';
import ThemeToggle from './components/ThemeToggle.vue';
</script>