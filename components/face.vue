<template>
<g>
  <polygon :points="points" :fill="faceup? 'limegreen' : 'palegreen'" fill-opacity="1" stroke="black"></polygon>
</g>
</template>

<script>
export default{
  props:['points','faceup']  
}
</script>
