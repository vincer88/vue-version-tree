<template>
  <svg :width="2 * radius + 6" :height="2 * radius + 6">
    <circle
      :cx="radius + 3"
      :cy="radius + 3"
      :r="radius"
      fill="purple"
      stroke="blue"
      stroke-width="3"
      fill-opacity="0.3"
    ></circle>
  </svg>
</template>

<script>
export default {
  data: () => ({
    radius: 10
  }),
  props: {}
};
</script>
