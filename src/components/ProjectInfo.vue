<template>
    <div class="border rounded-lg overflow-hidden">
      <div class="p-4 border-b bg-muted/30">
        <h2 class="font-semibold">Project Information</h2>
      </div>
      
      <div class="p-4 space-y-4">
        <div v-if="!selectedProject.type">
          <p class="text-muted-foreground text-sm">
            Select a project type in the chat to see information here.
          </p>
        </div>
        
        <div v-else>
          <div class="mb-4">
            <h3 class="text-sm font-medium mb-1">Project Type</h3>
            <Badge>{{ selectedProject.type }}</Badge>
          </div>
          
          <div class="mb-4">
            <h3 class="text-sm font-medium mb-1">Features</h3>
            <div class="flex flex-wrap gap-1">
              <Badge v-for="feature in selectedProject.features" :key="feature" variant="outline">
                {{ feature }}
              </Badge>
            </div>
          </div>
          
          <Separator />
          
          <div class="mt-4">
            <h3 class="text-sm font-medium mb-2">Deployment Options</h3>
            <div class="space-y-2">
              <div v-for="option in deploymentOptions" :key="option.name" 
                   class="flex items-center p-2 border rounded hover:bg-accent cursor-pointer">
                <div class="w-8 h-8 flex items-center justify-center mr-3 bg-primary/10 rounded">
                  <component :is="option.icon" class="h-4 w-4 text-primary" />
                </div>
                <div>
                  <p class="text-sm font-medium">{{ option.name }}</p>
                  <p class="text-xs text-muted-foreground">{{ option.description }}</p>
                </div>
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>
  </template>
  
  <script setup lang="ts">
  import { ref, computed } from 'vue';
  import { Badge } from '@/components/ui/badge';
  import { Separator } from '@/components/ui/separator';
  import { Cloud, Server, Globe, Github } from 'lucide-vue-next';
  
  // This would normally be connected to the chat state
  // For demo purposes, we'll use a static example
  const selectedProject = ref({
    type: 'Vue.js',
    features: ['Authentication', 'Dashboard', 'API Integration']
  });
  
  const deploymentOptions = [
    {
      name: 'Vercel',
      description: 'Simple deployment with Git integration',
      icon: Cloud
    },
    {
      name: 'Netlify',
      description: 'Automated builds from Git',
      icon: Globe
    },
    {
      name: 'Self-hosted',
      description: 'Deploy to your own server',
      icon: Server
    },
    {
      name: 'GitHub Pages',
      description: 'Free hosting for static sites',
      icon: Github
    }
  ];
  </script>