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
                <Button class="mt-3" size="sm" @click="downloadProject">
                  <Download class="h-4 w-4 mr-2" />
                  Download Project Files
                </Button>
              </div>
            </div>
            <p v-else v-html="formatMessage(message.content)"></p>
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
  
  interface ChatMessage {
    role: string;
    content: string;
  }
  
  const userInput = ref('');
  const messages = ref<Message[]>([]);
  const isTyping = ref(false);
  const chatContainer = ref<HTMLElement | null>(null);
  const currentStep = ref('intro');
  const chatHistory = ref<ChatMessage[]>([
    {
      role: "system",
      content: "You are a helpful AI Developer Assistant. You help users create complete, structured projects based on their requirements. Always be specific, helpful, and provide detailed explanations."
    }
  ]);
  
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
  
  const formatMessage = (text: string) => {
    // Convert line breaks to <br> tags
    return text.replace(/\n/g, '<br>');
  };

  const sendMessage = async () => {
    if (!userInput.value.trim() || isTyping.value) return;
    
    // Add user message
    const userMessage = userInput.value.trim();
    messages.value.push({
      content: userMessage,
      isUser: true
    });
    
    userInput.value = '';
    
    // Update chat history
    chatHistory.value.push({
      role: "user",
      content: userMessage
    });
    
    // Start typing indicator
    isTyping.value = true;
    
    try {
      if (currentStep.value === 'intro') {
        // Handle intro step differently to show project types
        messages.value.push({
          content: "Great! Please select one of the following project types to get started:",
          isUser: false,
          type: 'project-types'
        });
        currentStep.value = 'select-type';
        isTyping.value = false;
        return;
      }
      
      // Call Mistral AI API
      const response = await fetchMistralResponse();
      
      // Add assistant response to messages and chat history
      if (currentStep.value === 'requirements' && userMessage.toLowerCase().includes('generate') || 
          userMessage.toLowerCase().includes('create project')) {
        // If user is asking to generate or create a project after requirements
        const projectStructure = await generateProjectStructure(response);
        
        messages.value.push({
          content: "Thanks for providing those details! Based on your requirements, I've generated a project structure for you:",
          isUser: false,
          type: 'project-result',
          projectStructure: projectStructure
        });
        
        currentStep.value = 'result';
      } else {
        // Regular message response
        messages.value.push({
          content: response,
          isUser: false
        });
        
        // Update chat history with assistant's response
        chatHistory.value.push({
          role: "assistant",
          content: response
        });
      }
    } catch (error) {
      // Handle API error
      console.error("Error calling Mistral AI:", error);
      messages.value.push({
        content: "I'm sorry, there was an error processing your request. Please try again.",
        isUser: false
      });
    } finally {
      isTyping.value = false;
    }
  };
  
  const fetchMistralResponse = async () => {
    const MISTRAL_API_KEY = import.meta.env.VITE_MISTRAL_API_KEY;
    
    if (!MISTRAL_API_KEY) {
      console.error("Mistral API key is not defined");
      return "I'm sorry, but I can't process your request right now. Please make sure the API key is configured.";
    }
    
    try {
      const response = await fetch("https://api.mistral.ai/v1/chat/completions", {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
          "Authorization": `Bearer ${MISTRAL_API_KEY}`
        },
        body: JSON.stringify({
          model: "mistral-medium",
          messages: chatHistory.value,
          temperature: 0.7,
          max_tokens: 2000
        })
      });
      
      if (!response.ok) {
        throw new Error(`API request failed with status ${response.status}`);
      }
      
      const data = await response.json();
      return data.choices[0].message.content;
    } catch (error) {
      console.error("Error calling Mistral API:", error);
      throw error;
    }
  };
  
  const generateProjectStructure = async (response: string) => {
    // Ask Mistral AI to generate a project structure based on the requirements
    chatHistory.value.push({
      role: "assistant",
      content: response
    });
    
    chatHistory.value.push({
      role: "user",
      content: "Based on our conversation, please generate a detailed file structure for the project. Format it as a tree structure that I can display to the user."
    });
    
    try {
      const structureResponse = await fetchMistralResponse();
      
      // Extract the structure from the response
      const structureMatch = structureResponse.match(/```[\s\S]*?\n([\s\S]*?)```/);
      return structureMatch ? structureMatch[1].trim() : structureResponse;
    } catch (error) {
      console.error("Error generating project structure:", error);
      return `project-root/
  ├── src/
  │   ├── components/
  │   ├── views/
  │   ├── assets/
  │   └── main file
  ├── public/
  ├── package.json
  └── README.md`;
    }
  };
  
  const selectProjectType = (typeId: string) => {
    const selectedType = projectTypes.find(type => type.id === typeId);
    
    if (selectedType) {
      messages.value.push({
        content: `I want to create a ${selectedType.name} project`,
        isUser: true
      });
      
      // Add to chat history
      chatHistory.value.push({
        role: "user",
        content: `I want to create a ${selectedType.name} project`
      });
      
      // Simulate AI thinking
      isTyping.value = true;
      
      // Call Mistral API
      fetchMistralResponse()
        .then(response => {
          isTyping.value = false;
          messages.value.push({
            content: response,
            isUser: false
          });
          
          // Add response to chat history
          chatHistory.value.push({
            role: "assistant",
            content: response
          });
          
          currentStep.value = 'requirements';
        })
        .catch(error => {
          console.error("Error in selectProjectType:", error);
          isTyping.value = false;
          messages.value.push({
            content: `Great choice! ${selectedType.name} is an excellent framework for web development. To help me generate the best project structure for you, could you tell me more about your project? What kind of application are you building, and what features would you like to include?`,
            isUser: false
          });
          currentStep.value = 'requirements';
        });
    }
  };
  
  const downloadProject = () => {
    // This function would prepare project files for download
    // For now, it's just a placeholder
    alert("This feature is coming soon! Project download functionality will be implemented in a future update.");
  };
  
  const resetChat = () => {
    messages.value = [];
    currentStep.value = 'intro';
    
    // Reset chat history, keeping only the system message
    chatHistory.value = [
      {
        role: "system",
        content: "You are a helpful AI Developer Assistant. You help users create complete, structured projects based on their requirements. Always be specific, helpful, and provide detailed explanations."
      }
    ];
    
    // Add welcome message after a short delay
    setTimeout(() => {
      messages.value.push(welcomeMessage);
    }, 500);
  };
  </script>