<template>
    <SeparatorPrimitive
      :class="cn(
        'shrink-0 bg-border',
        orientation === 'horizontal' ? 'h-[1px] w-full' : 'h-full w-[1px]',
        className
      )"
      :orientation="orientation"
      :decorative="decorative"
    />
  </template>
  
  <script setup lang="ts">
  import { Separator as SeparatorPrimitive } from "radix-vue";
  import { cn } from "@/lib/utils";
  
  interface SeparatorProps {
    orientation?: "horizontal" | "vertical";
    decorative?: boolean;
    className?: string;
  }
  
  withDefaults(defineProps<SeparatorProps>(), {
    orientation: "horizontal",
    decorative: true,
    className: "",
  });
  </script>