<template>
  <div class="border rounded-lg overflow-hidden h-[600px] flex flex-col">
    <div class="p-3 border-b bg-muted/30">
      <h2 class="font-semibold text-sm">Project Information</h2>
    </div>
    
    <div class="p-3 space-y-3 overflow-y-auto flex-1">
      <div v-if="!projectDetails">
        <p class="text-muted-foreground text-sm">
          Select a project type in the chat to see information here.
        </p>
      </div>
      
      <div v-else>
        <div class="mb-3 flex items-center gap-2">
          <div class="w-8 h-8 rounded-full bg-primary/10 flex items-center justify-center">
            <component :is="getProjectIcon(projectDetails.type)" class="h-4 w-4 text-primary" />
          </div>
          <div>
            <h3 class="text-sm font-medium">{{ getProjectTypeName(projectDetails.type) }}</h3>
            <p class="text-xs text-muted-foreground">{{ getProjectStatus(projectDetails.status) }}</p>
          </div>
        </div>
        
        <div class="mb-3" v-if="currentProject.features && currentProject.features.length">
          <h3 class="text-xs font-medium mb-1 flex items-center">
            <ListChecks class="h-3.5 w-3.5 mr-1" /> Features
          </h3>
          <div class="flex flex-wrap gap-1">
            <Badge v-for="feature in currentProject.features" :key="feature" variant="outline" class="text-xs">
              {{ feature }}
            </Badge>
          </div>
        </div>
        
        <div v-if="currentProject.requirements && currentProject.requirements.length" class="mb-3">
          <h3 class="text-xs font-medium mb-1 flex items-center">
            <MessageSquare class="h-3.5 w-3.5 mr-1" /> Requirements
          </h3>
          <ul class="text-xs space-y-1 list-disc list-inside">
            <li v-for="(req, index) in currentProject.requirements" :key="index">{{ req }}</li>
          </ul>
        </div>
        
        <Separator />
        
        <div class="mt-3">
          <h3 class="text-xs font-medium mb-2 flex items-center">
            <Upload class="h-3.5 w-3.5 mr-1" /> Deployment Options
          </h3>
          <div class="space-y-1.5">
            <div v-for="option in getDeploymentOptions()" :key="option.name" 
                class="flex items-center p-1.5 border rounded hover:bg-accent cursor-pointer"
                :class="{'border-primary/50 bg-primary/5': selectedDeployment?.name === option.name}"
                @click="selectDeployment(option)">
              <div class="w-6 h-6 flex items-center justify-center mr-2 bg-primary/10 rounded">
                <component :is="option.icon" class="h-3.5 w-3.5 text-primary" />
              </div>
              <div>
                <p class="text-xs font-medium">{{ option.name }}</p>
                <p class="text-xs text-muted-foreground">{{ option.description }}</p>
              </div>
            </div>
          </div>
        </div>

        <div v-if="selectedDeployment" class="mt-3 p-2 bg-muted/50 rounded text-xs">
          <p class="font-medium mb-1">Deployment Steps for {{ selectedDeployment.name }}</p>
          <ol class="list-decimal list-inside space-y-1">
            <li v-for="(step, index) in selectedDeployment.steps" :key="index">{{ step }}</li>
          </ol>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, computed, watch, inject } from 'vue';
import { Badge } from '@/components/ui/badge';
import { Separator } from '@/components/ui/separator';
import { Cloud, Server, Globe, Github, Code, Database, FileCode, Package, 
         ListChecks, MessageSquare, Upload } from 'lucide-vue-next';

// Interface for the project data
interface ProjectData {
  type: string;
  features: string[];
  requirements: string[];
}

// Interface for deployment options
interface DeploymentOption {
  name: string;
  description: string;
  icon: any;
  steps: string[];
  suitableFor: string[];
}

// Interface for project details provided by parent
interface ProjectDetails {
  type: string;
  name: string;
  answers: string[];
  status: string;
}

// Try to inject project data from parent component
const injectedProjectType = inject<any>('selectedProjectType');
const injectedUserAnswers = inject<any>('userAnswers');
const projectDetails = inject<any>('projectDetails');

// Initialize current project data
const currentProject = computed(() => {
  if (!projectDetails?.value?.type) return { type: '', features: [], requirements: [] };
  
  const answers = projectDetails.value.answers || [];
  
  // Extract features from user answers (assuming second answer has features)
  const features = answers.length > 1 
    ? answers[1].split(',').map((f: string) => f.trim())
    : [];
  
  // Create requirements from user answers
  const requirements = answers.map((answer: string, index: number) => {
    // Skip the features answer since we're handling it separately
    if (index === 1) return null;
    
    // Create a meaningful label based on question index
    const prefix = index === 0 ? 'Purpose: ' : 
                  index === 2 ? 'Auth: ' :
                  index === 3 ? 'UI: ' : 'APIs: ';
    
    return prefix + answer;
  }).filter(Boolean) as string[]; // Remove the null entry and typecast
  
  return {
    type: getProjectTypeName(projectDetails.value.type),
    features,
    requirements
  };
});

const selectedDeployment = ref<DeploymentOption | null>(null);

// Helper function to get full project name from ID
const getProjectTypeName = (typeId: string): string => {
  const projectTypes: Record<string, string> = {
    'mern': 'MERN Stack',
    'vue': 'Vue.js',
    'react': 'React',
    'wordpress': 'WordPress Plugin',
    'laravel': 'Laravel',
    'django': 'Django'
  };
  
  return projectTypes[typeId] || typeId;
};

// Helper function to get status text
const getProjectStatus = (status: string): string => {
  const statusMap: Record<string, string> = {
    'intro': 'Starting project...',
    'select-type': 'Selecting project type...',
    'questions': 'Gathering requirements...',
    'generating': 'Generating project...',
    'result': 'Project generated'
  };
  
  return statusMap[status] || 'In progress';
};

// Get icon for project type
const getProjectIcon = (typeId: string) => {
  const iconMap: Record<string, any> = {
    'mern': Database,
    'vue': Code,
    'react': Code,
    'wordpress': Package,
    'laravel': FileCode,
    'django': FileCode
  };
  
  return iconMap[typeId] || Code;
};

// Dynamic deployment options based on project type
const getDeploymentOptions = () => {
  const allDeploymentOptions: DeploymentOption[] = [
    {
      name: 'Vercel',
      description: 'Simple deployment with Git integration',
      icon: Cloud,
      suitableFor: ['vue', 'react', 'mern'],
      steps: [
        'Create account on vercel.com',
        'Connect your GitHub repository',
        'Configure build settings',
        'Deploy your application'
      ]
    },
    {
      name: 'Netlify',
      description: 'Automated builds from Git',
      icon: Globe,
      suitableFor: ['vue', 'react', 'mern'],
      steps: [
        'Sign up for Netlify',
        'Connect to your Git repository',
        'Set build command and directory',
        'Deploy your site'
      ]
    },
    {
      name: 'Self-hosted',
      description: 'Deploy to your own server',
      icon: Server,
      suitableFor: ['mern', 'laravel', 'django', 'wordpress'],
      steps: [
        'Prepare your server (install dependencies)',
        'Set up web server (Nginx/Apache)',
        'Configure environment variables',
        'Upload and deploy your code',
        'Set up database if needed'
      ]
    },
    {
      name: 'GitHub Pages',
      description: 'Free hosting for static sites',
      icon: Github,
      suitableFor: ['vue', 'react'],
      steps: [
        'Create gh-pages branch in your repository',
        'Configure build to output to docs/ folder',
        'Push changes to GitHub',
        'Enable GitHub Pages in repository settings'
      ]
    },
    {
      name: 'Heroku',
      description: 'Platform as a service for web apps',
      icon: Cloud,
      suitableFor: ['mern', 'laravel', 'django'],
      steps: [
        'Create Heroku account',
        'Install Heroku CLI',
        'Create a new Heroku app',
        'Connect to your repository',
        'Configure environment variables',
        'Deploy your application'
      ]
    }
  ];

  // Filter options based on project type
  if (projectDetails?.value?.type) {
    return allDeploymentOptions.filter(option => 
      option.suitableFor.includes(projectDetails.value.type)
    );
  }
  
  return allDeploymentOptions;
};

const selectDeployment = (option: DeploymentOption) => {
  selectedDeployment.value = option;
};

// Reset selected deployment when project type changes
watch(projectDetails, () => {
  selectedDeployment.value = null;
}, { deep: true });
</script>