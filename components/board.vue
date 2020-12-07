<template>
<g>
<polygon :points="points" fill="gray"></polygon>
<cell v-for="p in coords" :a="p.a" :b="p.b" exp="50"></cell>
<piece v-for="p in conf" :a="p.a" :b="p.b" :n="p.n" exp="50"></piece>
</g>
</template>

<script>
import cell from '~/components/slot.vue'
import piece from '~/components/piece.vue'
export default{
    props: ['level','exp'],
    data(){
	return {
	    coords:[],
	    conf:[{a:0,b:0,n:3},{a:2,b:1,n:1}],
	    }
    },
    mounted(){
	let L=this.level;
	for(let a=-L;a<=L;a++){
	    for(let b=-L;b<=L;b++){
		if(a+b>=-L && a+b<=L){
		    this.coords.push({a:a,b:b});
		}
	    }
	}    
    },
    computed:{
	points(){
	    let pts=""
	    for(let i=0;i<6;i++){
		pts +=" "+(400+Math.cos(Math.PI/6+Math.PI/3*i)*(2*this.level+2)*this.exp)+","+
		    (400+Math.sin(Math.PI/6+Math.PI/3*i)*(2*this.level+2)*this.exp)//連結
		
	    }
	    return pts;
	}
    },
    components: {
	cell,
	piece
  }
}
</script>
