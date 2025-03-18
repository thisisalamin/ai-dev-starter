<template>
    <div class="flex flex-col h-[600px] border rounded-lg overflow-hidden">
      <div class="p-4 border-b bg-muted/30 flex items-center justify-between">
        <h2 class="font-semibold">Chat with AI Developer Assistant</h2>
        <Button variant="outline" size="sm" @click="resetChat">
          <RefreshCw class="h-4 w-4 mr-2" />
          Reset Chat
        </Button>
      </div>
      
      <div class="flex-1 overflow-y-auto p-4 space-y-4" ref="chatContainer">
        <div v-for="(message, index) in messages" :key="index" 
             :class="['flex', message.isUser ? 'justify-end' : 'justify-start']">
          <div :class="[
            'max-w-[80%] rounded-lg p-3',
            message.isUser ? 'bg-primary text-primary-foreground' : 'bg-muted'
          ]">
            <div v-if="!message.isUser && message.type === 'project-types'" class="space-y-2">
              <p>{{ message.content }}</p>
              <div class="grid grid-cols-2 gap-2 mt-2">
                <Button 
                  v-for="type in projectTypes" 
                  :key="type.id" 
                  variant="secondary" 
                  size="sm"
                  @click="selectProjectType(type.id)"
                >
                  {{ type.name }}
                </Button>
              </div>
            </div>
            <div v-else-if="!message.isUser && message.type === 'project-result'" class="space-y-2">
              <p>{{ message.content }}</p>
              <div class="mt-4 border rounded p-3 bg-card text-card-foreground">
                <h3 class="font-medium mb-2">Project Structure:</h3>
                <pre class="text-xs overflow-x-auto">{{ message.projectStructure }}</pre>
                <Button class="mt-3" size="sm">
                  <Download class="h-4 w-4 mr-2" />
                  Download Project Files
                </Button>
              </div>
            </div>
            <p v-else>{{ message.content }}</p>
          </div>
        </div>
        <div v-if="isTyping" class="flex justify-start">
          <div class="bg-muted max-w-[80%] rounded-lg p-3">
            <div class="flex space-x-2">
              <div class="w-2 h-2 rounded-full bg-primary animate-bounce"></div>
              <div class="w-2 h-2 rounded-full bg-primary animate-bounce" style="animation-delay: 0.2s"></div>
              <div class="w-2 h-2 rounded-full bg-primary animate-bounce" style="animation-delay: 0.4s"></div>
            </div>
          </div>
        </div>
      </div>
      
      <div class="p-4 border-t">
        <form @submit.prevent="sendMessage" class="flex gap-2">
          <Input 
            v-model="userInput" 
            placeholder="Type your message here..." 
            class="flex-1"
            :disabled="isTyping"
          />
          <Button type="submit" :disabled="!userInput.trim() || isTyping">
            <Send class="h-4 w-4" />
            <span class="sr-only">Send</span>
          </Button>
        </form>
      </div>
    </div>
  </template>
  
  <script setup lang="ts">
  import { ref, onMounted, watch, nextTick } from 'vue';
  import { Button } from '@/components/ui/button';
  import { Input } from '@/components/ui/input';
  import { RefreshCw, Send, Download } from 'lucide-vue-next';
  
  interface Message {
    content: string;
    isUser: boolean;
    type?: 'text' | 'project-types' | 'project-result';
    projectStructure?: string;
  }
  
  const userInput = ref('');
  const messages = ref<Message[]>([]);
  const isTyping = ref(false);
  const chatContainer = ref<HTMLElement | null>(null);
  const currentStep = ref('intro');
  
  const projectTypes = [
    { id: 'mern', name: 'MERN Stack' },
    { id: 'vue', name: 'Vue.js' },
    { id: 'react', name: 'React' },
    { id: 'wordpress', name: 'WordPress' },
    { id: 'laravel', name: 'Laravel' },
    { id: 'django', name: 'Django' }
  ];
  
  const welcomeMessage = {
    content: "👋 Hi there! I'm your AI Developer Assistant. I can help you generate complete, structured projects based on your ideas. What kind of project would you like to create today?",
    isUser: false,
    type: 'project-types'
  };
  
  onMounted(() => {
    // Add welcome message after a short delay
    setTimeout(() => {
      messages.value.push(welcomeMessage);
    }, 500);
  });
  
  watch(messages, () => {
    // Scroll to bottom when messages change
    nextTick(() => {
      if (chatContainer.value) {
        chatContainer.value.scrollTop = chatContainer.value.scrollHeight;
      }
    });
  }, { deep: true });
  
  const sendMessage = async () => {
    if (!userInput.value.trim() || isTyping.value) return;
    
    // Add user message
    messages.value.push({
      content: userInput.value,
      isUser: true
    });
    
    const userMessage = userInput.value;
    userInput.value = '';
    
    // Simulate AI thinking
    isTyping.value = true;
    
    // Process based on current step
    setTimeout(() => {
      isTyping.value = false;
      
      if (currentStep.value === 'intro') {
        messages.value.push({
          content: "Great! Please select one of the following project types to get started:",
          isUser: false,
          type: 'project-types'
        });
        currentStep.value = 'select-type';
      } else if (currentStep.value === 'requirements') {
        messages.value.push({
          content: "Thanks for providing those details! Based on your requirements, I've generated a project structure for you:",
          isUser: false,
          type: 'project-result',
          projectStructure: `project-root/
  ├── src/
  │   ├── components/
  │   │   ├── Header.vue
  │   │   ├── Footer.vue
  │   │   └── Dashboard.vue
  │   ├── views/
  │   │   ├── Home.vue
  │   │   ├── About.vue
  │   │   └── Contact.vue
  │   ├── router/
  │   │   └── index.js
  │   ├── store/
  │   │   └── index.js
  │   ├── assets/
  │   │   └── styles/
  │   ├── App.vue
  │   └── main.js
  ├── public/
  │   ├── index.html
  │   └── favicon.ico
  ├── package.json
  ├── README.md
  └── .gitignore`
        });
        currentStep.value = 'result';
      } else {
        // Generic response
        messages.value.push({
          content: "I understand you're interested in creating a project. Could you tell me more about your specific requirements? What features would you like to include?",
          isUser: false
        });
      }
    }, 1500);
  };
  
  const selectProjectType = (typeId: string) => {
    // Add user selection as a message
    const selectedType = projectTypes.find(type => type.id === typeId);
    
    if (selectedType) {
      messages.value.push({
        content: `I want to create a ${selectedType.name} project`,
        isUser: true
      });
      
      // Simulate AI thinking
      isTyping.value = true;
      
      // Respond after a delay
      setTimeout(() => {
        isTyping.value = false;
        messages.value.push({
          content: `Great choice! ${selectedType.name} is an excellent framework for web development. To help me generate the best project structure for you, could you tell me more about your project? What kind of application are you building, and what features would you like to include?`,
          isUser: false
        });
        currentStep.value = 'requirements';
      }, 1500);
    }
  };
  
  const resetChat = () => {
    messages.value = [];
    currentStep.value = 'intro';
    
    // Add welcome message after a short delay
    setTimeout(() => {
      messages.value.push(welcomeMessage);
    }, 500);
  };
  </script>