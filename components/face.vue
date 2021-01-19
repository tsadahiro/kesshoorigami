<template>
<g>
  <mask :id="'mask'+id">
    <polygon :points="points" fill="white"></polygon>
    <polygon v-for="c in covers" :points="c" fill="black" stroke="black" stroke-width="1"></polygon>
  </mask>
  <polygon :points="points" :fill="'palegreen'"
	   fill-opacity="1" stroke="black"
	   :mask="'url(#mask'+id+')'"
	   ></polygon>
</g>
</template>

<script>
export default{
    props:['id','points','faceup','covers','direction'],
    methods:{
    }
}
</script>
