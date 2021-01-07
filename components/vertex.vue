<template>
<g>
  <circle :cx="point.x" :cy="point.y" r="3" stroke="black" fill="red"></circle>
</g>
</template>

<script>
export default{
    props:['vertex'],

    computed:{
	exp(){
	    return this.$store.state.exp;
	},

	point(){
	    return {x:(this.vertex.x+1)*this.exp,
		    y:(this.vertex.y+1)*this.exp}
	},
    }
}
</script>
