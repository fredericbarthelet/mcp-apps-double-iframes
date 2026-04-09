<template>
  <div class="slidev-layout alpic-light-bg h-full relative overflow-hidden">
    <img src="/logos/alpic/logo-only-black.svg" class="logo-top-left" alt="" />
    <img
      src="/backgrounds/bicolor-montain.png"
      class="absolute bottom-0 right-0 w-3/5 object-contain object-right-bottom pointer-events-none"
      alt=""
    />
    <div class="relative z-10 h-full flex flex-col justify-center px-16">
      <slot />
    </div>
  </div>
</template>
