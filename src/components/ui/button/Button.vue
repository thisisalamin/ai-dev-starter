<template>
    <component
      :is="asChild ? Slot : 'button'"
      :class="buttonVariants({ variant, size, className })"
      :disabled="disabled || loading"
      v-bind="$attrs"
    >
      <Loader2 v-if="loading" class="mr-2 h-4 w-4 animate-spin" />
      <slot />
    </component>
  </template>
  
  <script setup lang="ts">
  import { Loader2 } from "lucide-vue-next";
  import { buttonVariants } from ".";
  import { Slot } from "radix-vue";
  
  interface ButtonProps {
    variant?: "default" | "destructive" | "outline" | "secondary" | "ghost" | "link";
    size?: "default" | "sm" | "lg" | "icon";
    asChild?: boolean;
    loading?: boolean;
    disabled?: boolean;
    className?: string;
  }
  
  withDefaults(defineProps<ButtonProps>(), {
    variant: "default",
    size: "default",
    asChild: false,
    loading: false,
    disabled: false,
    className: "",
  });
  </script>