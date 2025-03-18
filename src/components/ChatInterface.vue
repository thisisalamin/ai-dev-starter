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
            <div v-else-if="!message.isUser && message.type === 'project-result'" class="space-y-4">
              <p>{{ message.content }}</p>
              
              <!-- Project Structure -->
              <div class="mt-2 border rounded p-3 bg-card text-card-foreground">
                <h3 class="font-medium mb-2">Project Structure:</h3>
                <pre class="text-xs overflow-x-auto">{{ message.projectStructure }}</pre>
              </div>
              
              <!-- Key Files -->
              <div v-if="message.codeExamples && message.codeExamples.length" class="border rounded p-3 bg-card text-card-foreground">
                <h3 class="font-medium mb-2">Key Files:</h3>
                <div v-for="(example, i) in message.codeExamples" :key="i" class="mb-4">
                  <div class="flex justify-between items-center mb-1">
                    <h4 class="text-sm font-medium">{{ example.fileName }}</h4>
                    <Button variant="ghost" size="sm" @click="copyToClipboard(example.code)" class="h-6 text-xs">
                      <Copy class="h-3 w-3 mr-1" /> Copy
                    </Button>
                  </div>
                  <pre class="text-xs overflow-x-auto bg-muted p-2 rounded">{{ example.code }}</pre>
                </div>
              </div>
              
              <!-- Implementation Guide -->
              <div v-if="message.implementationGuide" class="border rounded p-3 bg-card text-card-foreground">
                <h3 class="font-medium mb-2">Implementation Guide:</h3>
                <div v-html="formatMessage(message.implementationGuide)" class="text-sm"></div>
              </div>
              
              <!-- Recommended Resources -->
              <div v-if="message.resources && message.resources.length" class="border rounded p-3 bg-card text-card-foreground">
                <h3 class="font-medium mb-2">Recommended Resources:</h3>
                <ul class="list-disc list-inside text-sm">
                  <li v-for="(resource, i) in message.resources" :key="i">
                    <a :href="resource.url" target="_blank" class="text-blue-500 hover:underline">{{ resource.title }}</a>
                    - {{ resource.description }}
                  </li>
                </ul>
              </div>
              
              <Button class="mt-3" size="sm" @click="downloadProject">
                <Download class="h-4 w-4 mr-2" />
                Download Project Files
              </Button>
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
  import { RefreshCw, Send, Download, Copy } from 'lucide-vue-next';
  
  interface CodeExample {
    fileName: string;
    code: string;
    description?: string;
  }
  
  interface Resource {
    title: string;
    url: string;
    description: string;
  }
  
  interface Message {
    content: string;
    isUser: boolean;
    type?: 'text' | 'project-types' | 'project-result';
    projectStructure?: string;
    codeExamples?: CodeExample[];
    implementationGuide?: string;
    resources?: Resource[];
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
  const currentQuestionIndex = ref(0);
  const selectedProjectType = ref('');
  const userAnswers = ref<string[]>([]);
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
  
  // Questions for each project type
  const projectQuestions = {
    default: [
      "What is the main purpose of your application?",
      "What specific features would you like to include?",
      "Do you need user authentication? If yes, what type (email/password, social logins)?",
      "Would you like a responsive design for mobile devices?",
      "Do you need any specific API integrations?"
    ],
    mern: [
      "What is the main purpose of your MERN application?",
      "What MongoDB collections/data models will you need?",
      "What Express routes and API endpoints should be included?",
      "What React components do you need for the frontend?",
      "Do you require user authentication or authorization?"
    ],
    vue: [
      "What is the main purpose of your Vue.js application?",
      "Would you prefer Vue 2 or Vue 3 for this project?",
      "Do you need state management (Vuex/Pinia)?",
      "What UI components or library would you like to use?",
      "Do you require any specific Vue plugins?"
    ],
    react: [
      "What is the main purpose of your React application?",
      "Would you prefer functional components with hooks or class components?",
      "Do you need state management (Redux, Context API, Recoil)?",
      "What UI framework would you like to use (Material UI, Chakra UI, etc.)?",
      "Do you need client-side routing with React Router?"
    ],
    wordpress: [
      "What type of WordPress plugin are you building? (e.g., admin tool, content editor, SEO, ecommerce, etc.)",
      "Will your plugin need custom database tables or will it use WordPress custom post types?",
      "What WordPress hooks (actions/filters) will your plugin need to integrate with?",
      "Will your plugin have an admin dashboard? If yes, what settings or features will it need?",
      "Does your plugin need to integrate with any external APIs or other WordPress plugins?"
    ],
    laravel: [
      "What is the main purpose of your Laravel application?",
      "What database models and relationships do you need?",
      "Do you need API endpoints or just web routes?",
      "Will you use Blade templates or a frontend framework with Laravel as API?",
      "Do you need user roles and permissions?"
    ],
    django: [
      "What is the main purpose of your Django application?",
      "What database models do you need?",
      "Will you use Django REST Framework for APIs?",
      "Do you need Django's built-in admin interface customized?",
      "What authentication system do you prefer?"
    ]
  };
  
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
      
      if (currentStep.value === 'questions') {
        // Store the user's answer
        userAnswers.value.push(userMessage);
        
        // If we've asked all questions, generate the project
        if (currentQuestionIndex.value >= 4) { // 0-based index, so 4 is the 5th question
          currentStep.value = 'generating';
          
          // Let the user know we're generating their project
          messages.value.push({
            content: "Thanks for answering all the questions! I'm now generating your project structure based on your requirements...",
            isUser: false
          });
          
          // Generate project structure with all the collected answers
          const projectPrompt = `Create a detailed and comprehensive project for a ${selectedProjectType.value} project with the following requirements:
  1. Main purpose: ${userAnswers.value[0]}
  2. Features: ${userAnswers.value[1]}
  3. ${userAnswers.value[2]}
  4. ${userAnswers.value[3]}
  5. ${userAnswers.value[4]}
  
  Please provide:
  1. A detailed file structure in tree format
  2. Example code for 3-5 key files that would help the developer get started
  3. Step-by-step implementation guide
  4. List of recommended libraries, packages, or dependencies needed
  5. Useful resources for development (documentation links, tutorials, etc.)`;
          
          // Add this detailed prompt to the chat history
          chatHistory.value.push({
            role: "user",
            content: projectPrompt
          });
          
          // Get response from API
          const response = await fetchMistralResponse();
          
          // Now generate the enhanced project output
          const projectOutput = await generateEnhancedProject(response);
          
          messages.value.push({
            content: "Based on your requirements, I've created a complete project blueprint for you:",
            isUser: false,
            type: 'project-result',
            projectStructure: projectOutput.structure,
            codeExamples: projectOutput.codeExamples,
            implementationGuide: projectOutput.implementationGuide,
            resources: projectOutput.resources
          });
          
          currentStep.value = 'result';
          isTyping.value = false;
          return;
        }
        
        // Ask the next question
        currentQuestionIndex.value++;
        const nextQuestion = getNextQuestion();
        messages.value.push({
          content: nextQuestion,
          isUser: false
        });
        
        isTyping.value = false;
        return;
      }
      
      // Call Mistral AI API for other cases
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
  
  // Get the next question based on project type and index
  const getNextQuestion = () => {
    const questions = projectQuestions[selectedProjectType.value as keyof typeof projectQuestions] || 
                     projectQuestions.default;
    return questions[currentQuestionIndex.value];
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
  
  // Enhanced project generation that extracts all the needed components
  const generateEnhancedProject = async (response: string) => {
    // Add the initial response to chat history
    chatHistory.value.push({
      role: "assistant",
      content: response
    });
    
    // Let's create a specific prompt for each component we need
    const structurePrompt = "Based on our conversation, please generate a detailed file structure for the project. Format it as a tree structure that I can display to the user.";
    const codeExamplesPrompt = "Please provide example code for 3-5 key files from the project. For each file, include the filename, the code content, and a brief description of what the file does. Format each example with a clear filename heading followed by the code block.";
    const guidePrompt = "Please provide a step-by-step implementation guide for this project. Include how to set up the environment, install dependencies, and implement the main features. Make it detailed but concise.";
    const resourcesPrompt = "Please suggest 3-5 important resources for this project, including official documentation, tutorials, libraries, and tools. For each resource, provide the title, URL, and a brief description.";
    
    // We'll extract project structure first
    chatHistory.value.push({
      role: "user",
      content: structurePrompt
    });
    
    const structureResponse = await fetchMistralResponse();
    const structure = extractProjectStructure(structureResponse);
    
    // Add this response and ask for code examples
    chatHistory.value.push({
      role: "assistant",
      content: structureResponse
    });
    chatHistory.value.push({
      role: "user",
      content: codeExamplesPrompt
    });
    
    const codeResponse = await fetchMistralResponse();
    const codeExamples = extractCodeExamples(codeResponse);
    
    // Add this response and ask for implementation guide
    chatHistory.value.push({
      role: "assistant",
      content: codeResponse
    });
    chatHistory.value.push({
      role: "user",
      content: guidePrompt
    });
    
    const guideResponse = await fetchMistralResponse();
    
    // Add this response and ask for resources
    chatHistory.value.push({
      role: "assistant",
      content: guideResponse
    });
    chatHistory.value.push({
      role: "user",
      content: resourcesPrompt
    });
    
    const resourcesResponse = await fetchMistralResponse();
    const resources = extractResources(resourcesResponse);
    
    // Return all components
    return {
      structure,
      codeExamples,
      implementationGuide: guideResponse,
      resources
    };
  };
  
  // Helper function to extract project structure from response
  const extractProjectStructure = (response: string) => {
    const structureMatch = response.match(/```[\s\S]*?\n([\s\S]*?)```/);
    return structureMatch ? structureMatch[1].trim() : response;
  };
  
  // Helper function to extract code examples from response
  const extractCodeExamples = (response: string): CodeExample[] => {
    const examples: CodeExample[] = [];
    
    // Look for code blocks with filename patterns
    const codeBlockRegex = /#+\s*(.*?)\s*\n```[\w]*\n([\s\S]*?)```/g;
    let match;
    
    while ((match = codeBlockRegex.exec(response)) !== null) {
      const fileName = match[1].trim();
      const code = match[2].trim();
      
      examples.push({
        fileName,
        code
      });
    }
    
    // If we couldn't find structured code blocks, try a simpler approach
    if (examples.length === 0) {
      const simpleCodeBlocks = response.match(/```[\w]*\n([\s\S]*?)```/g);
      if (simpleCodeBlocks) {
        simpleCodeBlocks.forEach((block, index) => {
          const code = block.replace(/```[\w]*\n|```/g, '');
          examples.push({
            fileName: `Example ${index + 1}`,
            code
          });
        });
      }
    }
    
    return examples;
  };
  
  // Helper function to extract resources from response
  const extractResources = (response: string): Resource[] => {
    const resources: Resource[] = [];
    
    // Try to detect resources in different formats
    // Format 1: Markdown links with descriptions
    const linkRegex = /\[(.*?)\]\((https?:\/\/[^\s)]+)\)\s*-\s*(.*?)(?=\n|$)/g;
    let match;
    
    while ((match = linkRegex.exec(response)) !== null) {
      resources.push({
        title: match[1],
        url: match[2],
        description: match[3].trim()
      });
    }
    
    // Format 2: Numbered/bulleted lists with titles and URLs
    if (resources.length === 0) {
      const listItemRegex = /(?:\d+\.|\*|\-)\s+(.*?):\s*(https?:\/\/[^\s]+)(?:\s*-\s*(.*?))?(?=\n|$)/g;
      
      while ((match = listItemRegex.exec(response)) !== null) {
        resources.push({
          title: match[1],
          url: match[2],
          description: match[3] ? match[3].trim() : 'Useful resource for this project'
        });
      }
    }
    
    return resources;
  };
  
  const copyToClipboard = (text: string) => {
    navigator.clipboard.writeText(text);
    // You could add a toast notification here
  };
  
  const generateProjectStructure = async (response: string) => {
    // This is now just a wrapper for backward compatibility
    // It calls our new enhanced project generator
    const projectOutput = await generateEnhancedProject(response);
    return projectOutput.structure;
  };
  
  const selectProjectType = (typeId: string) => {
    const selectedType = projectTypes.find(type => type.id === typeId);
    
    if (selectedType) {
      selectedProjectType.value = typeId;
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
      
      // Start the question sequence instead of generating right away
      setTimeout(() => {
        isTyping.value = false;
        messages.value.push({
          content: `Great choice! To help me create the perfect ${selectedType.name} project for you, I need to ask you 5 quick questions. Let's start with the first one:`,
          isUser: false
        });
        
        // Ask the first question
        currentStep.value = 'questions';
        currentQuestionIndex.value = 0;
        userAnswers.value = []; // Reset answers
        
        // Add the first question
        setTimeout(() => {
          messages.value.push({
            content: getNextQuestion(),
            isUser: false
          });
        }, 500);
      }, 1000);
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
    currentQuestionIndex.value = 0;
    selectedProjectType.value = '';
    userAnswers.value = [];
    
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