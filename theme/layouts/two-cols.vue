<template>
  <div
    class="slidev-layout alpic-dark-bg h-full relative overflow-hidden px-14 py-10"
  >
    <img src="/logos/alpic/logo-only-white.svg" class="logo-top-left" alt="" />
    <img
      src="/backgrounds/dotted-montain.png"
      class="absolute bottom-0 left-0 w-56 opacity-20 pointer-events-none"
      alt=""
    />
    <div class="relative z-10 h-full flex flex-col justify-center">
      <div class="mb-4">
        <slot name="default" />
      </div>
      <div class="grid grid-cols-2 gap-8 flex-1">
        <div><slot name="left" /></div>
        <div><slot name="right" /></div>
      </div>
    </div>
  </div>
</template>
