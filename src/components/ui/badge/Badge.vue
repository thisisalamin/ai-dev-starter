<template>
    <component
      :is="asChild ? Slot : 'div'"
      :class="badgeVariants({ variant, className })"
    >
      <slot />
    </component>
  </template>
  
  <script setup lang="ts">
  import { Slot } from "radix-vue";
  import { badgeVariants } from ".";
  
  interface BadgeProps {
    variant?: "default" | "secondary" | "destructive" | "outline";
    asChild?: boolean;
    className?: string;
  }
  
  withDefaults(defineProps<BadgeProps>(), {
    variant: "default",
    asChild: false,
    className: "",
  });
  </script>