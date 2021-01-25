<template>
<div>
  <svg width="800" height="600" :viewBox="view"
       @mousedown="mouseDown($event)"
       @mousemove="mouseMove($event)"
       @mouseup="mouseUp($event)"
       style="border:solid 1px #000000"
       >
    <g v-if="F">
    <face v-for="(f,i) in F" :id="i" :points="points(f)" :covers="coverOf(i)" :direction="directions && directions[i]" :faceup="faceup && faceup[i]"></face>
    <edge v-if="subdivided && !folded" v-for="(e,i) in E" :edge="edge(i)"></edge>
    <!--<circle v-for="f in F" :cx="mCenter(f).x*exp+exp" :cy="mCenter(f).y*exp+exp" r="3" fill="black"></circle>
    <line v-if="Tree && F" v-for="(f,i) in F" :x1="mCenter(f).x*exp+exp" :y1="mCenter(f).y*exp+exp" :x2="mCenter(F[Tree[i]]).x*exp+exp" :y2="mCenter(F[Tree[i]]).y*exp+exp" stroke="black"></line>-->
    </g>
  </svg><br/>
  <v-checkbox v-model="fromAbove" label="see from Above"> </v-checkbox>
  <v-btn @click="makeSquares(5,5)">squares</v-btn>
  <v-btn @click="makeTriangles(3)">triangle</v-btn>
  <v-btn @click="makeTriHex(3)">HexTri</v-btn>
  <v-btn v-if="!subdivided" @click="subdivide"> subdivide </v-btn>
  <v-btn v-if="subdivided && !folded" @click="foldAll()"> fold </v-btn>
  <v-btn v-if="subdivided && folded" @click="folded=false"> unfold </v-btn>
  <v-slider :label="'縮小:'+contract" v-model="contract" min="10" max="100"></v-slider>
  <v-slider :label="'回転:'+angle" v-model="angle" min="-90" max="90"></v-slider>
  <v-slider label="スケール" v-model="exp" min="50" max="200"></v-slider>
  {{cover}}
</div>
</template>

<script>
import face from "~/components/face.vue"
import edge from "~/components/edge.vue"
import vertex from "~/components/vertex.vue"

export default {
    data(){
	return {
	    subdivided: false,
            V:null,//頂点データ (x,y)
            E:null,//辺データ [from ,to]
            F:null,//面データ 辺のindexの集合
	    mountain:null, //山折り:true 谷折り:false
	    directions:null,//各面が順: true , 逆: false
	    numEdges:null,
	    faceup:null,//表向き: true, 裏向き: false,
	    cover:null,//上に来る面のindex
	    lower:null,//下に来る面のindex
	    fromAbove: true,
	    oV: null,// original vertices
	    oE: null,// original edges
	    oF: null,// original faces
	    contract: 100,
	    prevContract: 100,
	    angle: 0,
	    foldedV:null,
	    folded:false,
	    Tree:null,
	    isMouseDown: false,
	    view:"0 0 800 600",
	    viewX: 0,
	    viewY: 0,
	    prevX: 0,
	    prevY: 0,
	    ratio: 1.0,
	    numPrimal: null,
	    numDual: null,
	    rPrimal: null,
	    rDual: null,
	}
    },
    watch:{
	contract:function(){
	    if (!this.subdivided) return;
	    let c=this.contract/100;
	    for (let i in this.oF){
		let vset = []
		for (let e of this.oF[i]){
		    for (let v of this.E[e]){
			if (vset.find((e)=>{e==v})===undefined){
			    vset.push(v)
			}
		    }
		}
		let sum = {x:0,y:0}
		for (let v of vset){
		    sum.x += this.V[v].x
		    sum.y += this.V[v].y
		}
		let m={x:sum.x/vset.length, y:sum.y/vset.length}
		/*
		for (let v of vset){
		    let vertex = {x: c*this.oV[v].x + (1-c)*m.x,
				  y: c*this.oV[v].y + (1-c)*m.y}
		    let theta = this.directions[i]*Math.PI*this.angle/180
		    vertex = {x: (vertex.x-m.x)*Math.cos(theta) - (vertex.y-m.y)*Math.sin(theta) + m.x,
			      y: (vertex.x-m.x)*Math.sin(theta) + (vertex.y-m.y)*Math.cos(theta) + m.y}
		    this.V.splice(v,1,vertex)
		}
		*/
		for (let v of vset){
		    //let vertex = {x: c*this.oV[v].x + (1-c)*m.x,
		    //		  y: c*this.oV[v].y + (1-c)*m.y}
		    let vertex = {x: this.oV[v].x,  y: this.oV[v].y}
		    let theta = Math.PI*this.angle/180
		    if (this.directions[i] == 1){
			vertex = {x: c*(vertex.x-m.x)*Math.cos(theta) - c*(vertex.y-m.y)*Math.sin(theta) + m.x,
				  y: c*(vertex.x-m.x)*Math.sin(theta) + c*(vertex.y-m.y)*Math.cos(theta) + m.y}
			this.V.splice(v,1,vertex)
		    }
		    else{
			console.log(this.rPrimal,this.rDual)
			let beta = Math.atan(Math.tan(Math.PI/this.numDual)/Math.tan(Math.PI/this.numPrimal)*Math.tan(theta))
			let R = 1;
			if (Math.abs(beta)<0.0001){
			    R = c
			}
			else{
			    R = c*this.rPrimal * Math.cos(Math.PI/this.numPrimal)*Math.sin(theta)/(Math.cos(Math.PI/this.numDual)*Math.sin(beta))/this.rDual
			}
			console.log(beta,R)
			vertex = {x: c*((vertex.x-m.x)*Math.cos(-beta) - (vertex.y-m.y)*Math.sin(-beta)) + m.x,
				  y: c*((vertex.x-m.x)*Math.sin(-beta) + (vertex.y-m.y)*Math.cos(-beta)) + m.y}
			this.V.splice(v,1,vertex)
		    }
		}
	    }
	    if (this.folded){
		this.foldAll();
	    }
	},
	angle(){
	    if (!this.subdivided) return;
	    //for (let f of this.oF){
	    let c=this.contract/100;
	    for (let i in this.oF){
		let vset = []
		for (let e of this.oF[i]){
		    for (let v of this.E[e]){
			if (vset.find((e)=>{e==v})===undefined){
			    vset.push(v)
			}
		    }
		}
		let sum = {x:0,y:0}
		for (let v of vset){
		    sum.x += this.V[v].x
		    sum.y += this.V[v].y
		}
		let m={x:sum.x/vset.length, y:sum.y/vset.length}
		for (let v of vset){
		    //let vertex = {x: c*this.oV[v].x + (1-c)*m.x,
		    //		  y: c*this.oV[v].y + (1-c)*m.y}
		    let vertex = {x: this.oV[v].x,  y: this.oV[v].y}
		    let theta = Math.PI*this.angle/180
		    if (this.directions[i] == 1){
			vertex = {x: c*(vertex.x-m.x)*Math.cos(theta) - c*(vertex.y-m.y)*Math.sin(theta) + m.x,
				  y: c*(vertex.x-m.x)*Math.sin(theta) + c*(vertex.y-m.y)*Math.cos(theta) + m.y}
			this.V.splice(v,1,vertex)
		    }
		    else{
			//let beta = -Math.acos(Math.cos(theta))
			//console.log(this.numPrimal,this.numDual)
			console.log(this.rPrimal,this.rDual)
			let beta = Math.atan(Math.tan(Math.PI/this.numDual)/Math.tan(Math.PI/this.numPrimal)*Math.tan(theta))
			let R = c*this.rPrimal * Math.cos(Math.PI/this.numPrimal)*Math.sin(theta)/(Math.cos(Math.PI/this.numDual)*Math.sin(beta))
			console.log(beta,R)
			vertex = {x: R*((vertex.x-m.x)*Math.cos(-beta) - (vertex.y-m.y)*Math.sin(-beta))/this.rDual + m.x,
				  y: R*((vertex.x-m.x)*Math.sin(-beta) + (vertex.y-m.y)*Math.cos(-beta))/this.rDual + m.y}
			this.V.splice(v,1,vertex)
		    }
		}
	    }
	    if (this.folded){
		this.foldAll();
	    }
	}
    },
    
    computed:{
	exp:{
	    get(){
		return this.$store.state.exp;
	    },
	    set(exp){
		this.$store.commit('setExp',exp);
	    }
	},
	adjList(){ // 面と面が辺を共有するとき、隣接すると考える。
	    let A = [];
	    for (let i in this.F){
		let R = [];
		for (let j in this.F){
		    if (this.isAdjacent(i,j)){
			R.push(j);
		    }
		}
		A.push(R);
	    }
	    return A;
	},
    },
    
    methods:{
	mouseDown(e){
	    console.log(e.offsetX + "," + e.offsetY);
	    this.isMouseDown = true;
	    this.prevX = e.offsetX;//動く前
	    this.prevY = e.offsetY; 
	},
	mouseUp(e){
	    console.log(e.offsetX + "," + e.offsetY);
	    this.isMouseDown = false;//マウスの押し下げが終わった
	},
	mouseMove(e){
	    if(this.isMouseDown){
		let deltaX = e.offsetX - this.prevX;//e.offsetは動いた後の場所
		let deltaY = e.offsetY - this.prevY;
		this.viewX += deltaX;
		this.viewY += deltaY;
		this.view = (-this.viewX) + " " + (-this.viewY) + " 800 800";
		this.prevX = e.offsetX;
		this.prevY = e.offsetY;
	    }
	},
	x1(e){
	    if(this.folded){
		return this.foldedV[e[0]].x*this.exp+this.exp;
	    }
	    else{
		return this.V[e[0]].x*this.exp+this.exp;
	    }
	},
	y1(e){
	    if(this.folded){
		return this.foldedV[e[0]].y*this.exp+this.exp;
	    }
	    else{
		return this.V[e[0]].y*this.exp+this.exp;
	    }
	},
	x2(e){
	    if(this.folded){
		return this.foldedV[e[1]].x*this.exp+this.exp;
	    }
	    else{
		return this.V[e[1]].x*this.exp+this.exp;
	    }
	},
	y2(e){
	    if (this.folded){
		return this.foldedV[e[1]].y*this.exp+this.exp;
	    }
	    else{
		return this.V[e[1]].y*this.exp+this.exp;
	    }
	},
	
	edge(i){
	    const from  = {x:this.V[this.E[i][0]].x, y: this.V[this.E[i][0]].y};
	    const to  = {x:this.V[this.E[i][1]].x, y: this.V[this.E[i][1]].y};
	    let color = null;
	    if (this.mountain && this.mountain[i]){
		color = "blue";
	    }
	    else{
		color = "red";
	    }
	    //console.log({from:from, to:to, color:color});
	    return {from:from, to:to, color:color};
	},
	
	dist(p,q){
	    return Math.sqrt((p.x-q.x)*(p.x-q.x) + (p.y-q.y)*(p.y-q.y));
	},
	
	points(f){
	    let seq = []
	    let pts = ""
	    seq.push(this.E[f[0]][0])
	    seq.push(this.E[f[0]][1])
	    let used = []
	    for (let e in this.E){
		used.push(false)
	    }
	    used[f[0]] = true;
	    pts = pts + " " + this.x1(this.E[f[0]]) + "," + this.y1(this.E[f[0]])
	    
	    while (seq.length <= f.length){
		for (let eidx of f){
		    if (used[eidx]) continue;
		    
		    let last = seq[seq.length-1]
		    if (this.E[eidx][0]==last && this.E[eidx][1]!=last){
			used[eidx] = true;
			seq.push(this.E[eidx][1])
			pts = pts + " " + this.x1(this.E[eidx]) + "," + this.y1(this.E[eidx])
			break;
		    }
		    if (this.E[eidx][1]==last && this.E[eidx][0]!=last){
			used[eidx] = true;
			seq.push(this.E[eidx][0])
			pts = pts + " " + this.x2(this.E[eidx]) + "," + this.y2(this.E[eidx])
			break;
		    }
		}
	    }
	    return pts;
	},
	
	coverOf(fidx){
	    if (this.fromAbove && this.cover ){
		let coverPoints = [];
		for (let c of this.cover[fidx]){
		    coverPoints.push(this.points(this.F[c]))
		}
		return coverPoints;
	    }
	    else if (this.lower){
		let coverPoints = [];
		for (let c of this.lower[fidx]){
		    coverPoints.push(this.points(this.F[c]))
		}
		return coverPoints;
	    }
	    else{
		return [];
	    }
	},
	lowerCoverOf(fidx){
	    if (this.cover){
		let coverPoints = [];
		for (let c of this.lower[fidx]){
		    coverPoints.push(this.points(this.F[c]))
		}
		return coverPoints;
	    }
	    else{
		return [];
	    }
	},
	directFaces(){
	    let directions =[]
	    let directed = []
	    let boundary = [0]
	    
	    for (let f in this.F){
		directed.push(false)
		directions.push(1)
	    }
	    directed[0]=true
	    this.numPrimal = this.F[0].length;
	    
	    let newBoundary = []
	    while(boundary.length > 0){
		console.log(boundary)
		for (let b of boundary){
		    for (let n of this.adjList[b]){
			if (!directed[n]){
			    directions[n] = -directions[b]
			    newBoundary.push(1*n)
			    directed[n]=true
			}
		    }
		}
		boundary = newBoundary.slice(0,newBoundary.length);
		newBoundary = []
	    }
	    for (let i in directions){
		if (directions[i]==-1){
		    this.numDual = this.F[i].length
		    break;
		}
	    }
	    this.directions = directions;
	    console.log("directed")
	},
	
	subdivide(){
	    let F = []
	    let E = []
	    let V = []
	    let mountain = []
	    let ancF = []
	    let ancE = []
	    let ancV = []
	    let vFaces = []
	    for (let v in this.V){
		vFaces.push([])
	    }
	    this.directFaces()
	    for (let f of this.F){
		let findices = []
		let count = 0
		for (let e of f){
		    let edge = []
		    for (let v of this.E[e]){
			if (findices[v] === undefined){
			    findices[v]=V.length
			    V.push(this.V[v])
			    ancV.push(v)
			    count++
			}
			edge.push(findices[v])
		    }
		    E.push(edge)
		    mountain.push(true);
		    ancE.push(1*e)
		}
		let face = []
		for (let i=0; i<f.length; i++){
		    face.push(E.length-f.length+i)
		}
		F.push(face)
		ancF.push(f)
	    }
	    //console.log(F.length, E.length, V.length)
	    this.oV = V.slice(0,V.length);
	    this.oE = E.slice(0,E.length);
	    this.oF = F.slice(0,F.length);
	    //console.log(ancE)
	    for (let eidx in this.E){//オリジナルのセル分割の各辺について面を作る
		// オリジナルの各辺はそれが２面に共有されたものであったら
		// 上の操作で２つに分裂しており、
		// fidx,sidxにはそれぞれの新しいEにおけるindexが入る。
		let fidx = ancE.indexOf(1*eidx)
		let sidx = ancE.lastIndexOf(1*eidx)

		if (fidx!=sidx){
		    let fFaceIndex = F.findIndex((f)=>(f.indexOf(fidx)>=0))
		    let sFaceIndex = F.findIndex((f)=>(f.indexOf(sidx)>=0))
		    //console.log(fFaceIndex,sFaceIndex)
		    //console.log(fFaceIndex, ":", this.directions[fFaceIndex], sFaceIndex, ":", this.directions[sFaceIndex]);
		    let vec1 = {x:this.V[this.E[eidx][1]].x - this.V[this.E[eidx][0]].x,
				y:this.V[this.E[eidx][1]].y - this.V[this.E[eidx][0]].y}
		    let mcenter = this.mCenter(this.F[fFaceIndex]);
		    let vec2 = {x:mcenter.x - this.V[this.E[eidx][0]].x,
				y:mcenter.y - this.V[this.E[eidx][0]].y}
		    let leftside = vec1.x*vec2.y-vec1.y*vec2.x;
		    let edge0 = [E[fidx][0], E[sidx][0]]
		    let edge1 = [E[fidx][1], E[sidx][1]]
		    //console.log(vec1,vec2);
		    //console.log(leftside);
		    //console.log("newedge", ancV[E[fidx][0]], ancV[E[fidx][1]])
		    E.push(edge0)
		    mountain.push(this.directions[fFaceIndex]*leftside > 0 ? true : false);
		    E.push(edge1)
		    mountain.push(this.directions[fFaceIndex]*leftside > 0 ? false : true);
		    vFaces[ancV[E[fidx][0]]].push(E.length-2)
		    vFaces[ancV[E[fidx][1]]].push(E.length-1)
		    F.push([fidx, sidx, E.length-2, E.length-1])
		}
	    }

	    //console.log(vFaces.filter((x)=>x.length>2))
	    //console.log(F)
	    this.V = V;
	    this.E = E;
	    this.F = F.concat(vFaces.filter((x)=>x.length>2)); // ここ変です。
	    this.mountain = mountain;

	    this.rPrimal = this.dist(this.mCenter(this.oF[0]), this.oV[this.oE[this.oF[0][0]][0]]);
	    console.log(this.rPrimal);
	    console.log(this.rDual);
	    for (let i in this.oF){
		if (this.directions[i]==-1){
		    this.rDual = this.dist(this.mCenter(this.oF[i]), this.oV[this.oE[this.oF[i][0]][0]]);
		    //console.log(this.rDual);
		    break;
		}
	    }
	    /*
	    let alpha = 0;
	    let beta = 0;
	    for (let fidx in  this.F){
		let f = this.F[fidx]
		if (this.directions[fidx]==1){
		    alpha = this.lenProj(this.mCenter(f), this.E[f[0]])
		    //console.log(alpha)
		    break;
		}
	    }
	    for (let fidx in  this.F){
		let f = this.F[fidx]
		if (this.directions[fidx]==-1){
		    beta = this.lenProj(this.mCenter(f), this.E[f[0]])
		    //console.log(beta)
		    break;
		}
	    }
	    this.ratio = alpha/beta;
	    console.log("ratio:", this.ratio);
	    */
	    //console.log(this.Tree)
	    this.Tree = this.makeTree();
	    this.subdivided = true
	    this.makeCoverRelation();
	    //console.log(this.directions)
	},

	lenProj(p,e){
	    console.log(p,e)
	    let q = {x:(this.V[e[0]].x+this.V[e[1]].x)/2, y:(this.V[e[0]].y+this.V[e[1]].y)/2};
	    return this.dist(p,q);
	},

	mCenter(f){
	    const V = []
	    let sum = {x:0,y:0}
	    for (let e of f){
		for (let v of this.E[e]){
		    if (V.indexOf(v)<0){
			V.push(v)
			sum.x += this.V[v].x
			sum.y += this.V[v].y
			//console.log(v,this.V[v].x, this.V[v].y)
		    }
		}
	    }
	    //console.log(V.length)
	    return {x: sum.x/V.length, y: sum.y/V.length}
	},
	
	isAdjacent: function(i,j){
	    if (i==j){
		return false
	    }
	    for (let e of this.F[i]){
		if (this.F[j].indexOf(e) > -1){
		    return true;
		}
	    }
	    return false;
	},

	commonEdge: function(i,j){
	    if (i==j){
		return -1
	    }
	    for (let e of this.F[i]){
		let cidx = this.F[j].indexOf(e)
		if (cidx > -1){
		    return e;
		}
	    }
	    return -1;
	},
	
	ref(p,e){ // p ={x:, y:}, e is the index of E
	    console.log(p,e);
	    const ax = this.V[this.E[e][0]].x;
	    const ay = this.V[this.E[e][0]].y;
	    const bx = this.V[this.E[e][1]].x;
	    const by = this.V[this.E[e][1]].y;
	    const Tp = {x:p.x,y:p.y}

	    const normSquared = (ax-bx)*(ax-bx)+(ay-by)*(ay-by)
	    const dotProd = (p.x-ax)*(-by+ay)+(p.y-ay)*(bx-ax)

	    Tp.x -= 2*dotProd/normSquared*(-by+ay)
	    Tp.y -= 2*dotProd/normSquared*(bx-ax)

	    return Tp
	},
	
	foldAll(){
	    let V = [];
	    this.foldedV = [];
	    for (let i in this.V){
		this.foldedV.push(null);
	    }
	    for (let f in this.F){
		//console.log(this.foldToRoot(f));
		let foldedImg = this.foldToRoot(f);
		console.log(foldedImg)
		for (let i in foldedImg.index){
		    this.foldedV[foldedImg.index[i]] = foldedImg.points[i];
		}
		console.log(this.V.length, this.foldedV.length);
	    }
	    this.folded=true;
	},

	foldToRoot(findex){
	    if (!this.subdivided) return;
	    
	    let isRoot = function(v,tree){
		if (tree[v] == v){
		    return true;
		}
		else{
		    return false;
		}
	    };
	    
	    let vset = []
	    for (let e of this.F[findex]){
		if (vset.indexOf(this.E[e][0])<0){
		    vset.push(this.E[e][0])
		}
		if (vset.indexOf(this.E[e][1])<0){
		    vset.push(this.E[e][1])
		}
	    }
	    let points = []
	    for (let v of vset){
		points.push(this.V[v])
	    }
	    while (!isRoot(findex,this.Tree)){
		console.log("face:",findex, this.Tree[findex])
		//console.log(points)
		let to = this.Tree[findex]
		let edge = null
		for (let e of this.F[findex]){
		    if (this.F[to].indexOf(e)>=0){
			edge = e
			break
		    }
		}
		let newPoints = []
		for (let p of points){
		    newPoints.push(this.ref(p,edge))
		    //console.log(this.V[v], " |--> ", this.ref(this.V[v],edge))
		}
		points = newPoints;
		findex = this.Tree[findex]
	    }
	    console.log("folded:",points)
	    return {index:vset,points:points}
	},

	    
	
	makeTree(){
	    let tree =[]
	    let inTree = []
	    let centers = []
	    let sum = {x:0,y:0}

	    for (let f in this.F){
		inTree.push(false)
		tree.push(0)
		centers.push(this.mCenter(f));
		sum.x += this.mCenter(f).x;
		sum.y += this.mCenter(f).y;
	    }
	    const totalCenter = {x:(sum.x/(this.F.length)), y:(sum.y/(this.F.length))}
	    console.log("total center:",totalCenter)
	    let rindex = 0;
	    console.log(centers[rindex]);
	    let minDist = this.dist(centers[rindex],totalCenter);
	    console.log(minDist);
	    for (let c in centers){
		if (this.dist(centers[c],totalCenter)<minDist){
		    rindex = 1*c;
		}
	    }
	    inTree[rindex]=true
	    let boundary = [rindex]
	    tree[rindex]=rindex;
	    this.faceup = [];
	    let faceup = true;
	    this.faceup[rindex] = faceup;

	    let newBoundary = []
	    while(boundary.length > 0){
		faceup = !faceup;
		for (let b of boundary){
		    for (let n of this.adjList[b]){
			if (!inTree[n]){
			    tree[n] = b
			    this.faceup[n] = faceup;
			    newBoundary.push(1*n)
			    inTree[n]=true
			}
		    }
		}
		boundary = newBoundary.slice(0,newBoundary.length);
		newBoundary = []
	    }
	    console.log("tree",tree)
	    return tree
	},

	makeCoverRelation(){
	    let cover = [];
	    let lower = [];
	    for (let i in this.F){
		let C = [];
		let L = [];
		for (let j in this.F){
		    let common = this.commonEdge(i,j)
		    if (common >= 0){
			if ((this.faceup[i] && this.mountain[common]) || (!this.faceup[i] && !this.mountain[common])){
			    C.push(j);
			}
			else{
			    L.push(j);
			}
		    }
		}
		cover.push(C);
		lower.push(L);
	    }
	    this.cover=cover;
	    this.lower=lower;
	    console.log("cover",cover)
	    console.log("lower",lower);
	},

	init(){
	    this.V=null;
	    this.E=null;
	    this.F=null;
	    this.mountain = null;
	    this.faceup = null;
	    this.directions = null;
	    this.subdivided = false;
	    this.folded = false;
	    this.cover=null;
	    this.lower=null;
	    this.fromAbove=true;
	    this.oV = null,// original vertices
	    this.oE = null,// original edges
	    this.oF = null,// original faces
	    this.contract = 100;
	    this.angle = 0;
	},
	
	makeSquares(m,n){
	  this.init();  

	  let mdist = function(p,q){
	      return Math.abs(p.x-q.x) + Math.abs(p.y-q.y)
	  };
	  
	  let V = [];
	  for (let i = 0; i < n; i++){
	      for(let j = 0; j < m; j++){
		  V.push({x:i, y:j})
	      }
	  }
	  let E = [];
	  for (let p in V){
	      for (let q in V){
		  if (mdist(V[p],V[q]) == 1 && (V[p].x <= V[q].x) && (V[p].y <= V[q].y)){
		      E.push([p,q])
		  }
	      }
	  }
	  
	  let F = [];

	  for (let i=0; i< n-1; i++){
	      for (let j=0; j<m-1; j++){
		  let f = []
		  let center = {x:(i+0.5), y:(j+0.5)}
		  for (let e in E){
		      if (mdist(center, V[E[e][0]]) <= 1 &&  mdist(center, V[E[e][1]]) <= 1){
			  f.push(e)
			  //console.log(center)
			  //console.log(V[E[e][0]])
			  //console.log(V[E[e][1]])
		      }
		  }
		  F.push(f)
	      }
	  }
	  this.V = V;
	  this.E = E;
	  this.F = F;

	  console.log(V);
	  console.log(E);
	  console.log(F);
	  console.log(V.length);
	  console.log(E.length);
	  console.log(F.length);
      },

	makeTriHex(n){
	    this.init();
	    const V=[]
	    const coefs = []
	    for (let a=-n; a<=n; a++){
		for (let b=-n; b<=n; b++){
		    if (Math.abs(a+b)<=n && !(Math.abs(a%2)==0 && Math.abs(b%2)==0)){
			V.push({x: a+b/2, y: b*Math.sqrt(3)/2})
			coefs.push({a:a, b:b})
		    }
		}
	    }
	    const E=[]
	    
	    for (let p in coefs){
		//for (let q=p+1; q < coefs.length; q++){
		for (let q in coefs){
		    if (p >= q) continue;
		    if (Math.abs(coefs[p].a-coefs[q].a)==1 && coefs[p].b == coefs[q].b){
			E.push([1*p,1*q]);
		    }
		    if (Math.abs(coefs[p].b-coefs[q].b)==1 && coefs[p].a == coefs[q].a){
			E.push([1*p,1*q]);
		    }
		    if (coefs[p].b-coefs[q].b == 1 && coefs[q].a - coefs[p].a == 1){
			E.push([1*p,1*q]);
		    }
		    if (coefs[p].b-coefs[q].b == -1 && coefs[q].a - coefs[p].a == -1){
			E.push([1*p,1*q]);
		    }
		}
	    }
	    //console.log(E)

	    const F = [];
	    for (let a=-n; a<=n; a++){
		for (let b=-n; b<=n; b++){
		    if (Math.abs(a+b)<=n && (Math.abs(a%2)==0 && Math.abs(b%2)==0)){
			const surrounding = [{a:1,b:0}, {a:0,b:1}, {a:-1,b:1}, {a:-1,b:0}, {a:0,b:-1}, {a:1,b:-1}]
			const face = []
			for (let i =0; i<6; i++){
			    //console.log(a,b,":",i)
			    let p = {a: (1*surrounding[i].a + 1*a), b: (1*surrounding[i].b + 1*b)}
			    let q = {a: (1*surrounding[(i+1)%6].a + 1*a), b: (1*surrounding[(i+1)%6].b + 1*b)}
			    let v = {x: 1*p.a+p.b/2, y: 1*p.b*Math.sqrt(3)/2}
			    let w = {x: 1*q.a+q.b/2, y: 1*q.b*Math.sqrt(3)/2}
			    //console.log(surrounding[i],i)
			    //console.log(surrounding[(i+1)%6], (i+1)%6)
			    //console.log(p,q)
			    //console.log({x:w.x-v.x, y:w.y-v.y})
			    let vidx = V.findIndex( g => Math.sqrt((v.x-g.x)*(v.x-g.x) + (v.y-g.y)*(v.y-g.y)) < 0.01)
			    let widx = V.findIndex( g => Math.sqrt((w.x-g.x)*(w.x-g.x) + (w.y-g.y)*(w.y-g.y)) < 0.01)
			    let eidx = -1
			    if (vidx >= 0 && widx >= 0){
				let s = E.findIndex( e => (e[0]==vidx && e[1]==widx) )
				let t = E.findIndex( e => e[0]==widx && e[1]==vidx)
				if (s >= 0){
				    eidx = s
				}
				else if (t >= 0){
				    eidx = t
				}
			    }
			    if (eidx >= 0){
				face.push(eidx);
				//console.log(face)
			    }
			}
			if (face.length == 6){
			    F.push(face)
			}
		    }
		}
	    }


	    for (let a in E){
		for (let b in E){
		    if (a >= b) continue
		    if (E[a][1] == E[b][0] ){
			let min = Math.min(E[a][0],E[b][1])
			let max = Math.max(E[a][0],E[b][1])
			for (let c in E){
			    if ((E[c][0]==min && E[c][1]==max) || (E[c][1]==min && E[c][0]==max)){
				F.push([1*a,1*b,1*c]);
			    }
			}
		    }
		    if (E[a][0] == E[b][1] ){
			let min = Math.min(E[a][1],E[b][0])
			let max = Math.max(E[a][1],E[b][0])
			for (let c in E){
			    if ((E[c][0]==min && E[c][1]==max) || (E[c][1]==min && E[c][0]==max)){
				F.push([1*a,1*b,1*c]);
			    }
			}
		    }
		}
	    }
	    //console.log(F)
	    this.V = V;
	    this.E = E;
	    this.F = F;
	    console.log(this.F);
	    console.log(this.E);
	    console.log(this.adjList);
	},

	makeTriangles(n){
	    this.init();
	    const V=[]
	    const coefs = []
	    for (let a=-n; a<=n; a++){
		for (let b=-n; b<=n; b++){
		    if (Math.abs(a+b)<=n){
			V.push({x: a+b/2, y: b*Math.sqrt(3)/2})
			coefs.push({a:a, b:b})
		    }
		}
	    }
	    const E=[]
	    for (let p in coefs){
		for (let q in coefs){
		    if (p >= q) continue;
		    if (Math.abs(coefs[p].a-coefs[q].a)==1 && coefs[p].b == coefs[q].b){
			E.push([1*p,1*q]);
		    }
		    if (Math.abs(coefs[p].b-coefs[q].b)==1 && coefs[p].a == coefs[q].a){
			E.push([1*p,1*q]);
		    }
		    if (coefs[p].b-coefs[q].b == 1 && coefs[q].a - coefs[p].a == 1){
			E.push([1*p,1*q]);
		    }
		    if (coefs[p].b-coefs[q].b == -1 && coefs[q].a - coefs[p].a == -1){
			E.push([1*p,1*q]);
		    }
		}
	    }
	    console.log(E)
	    const F = [];
	    for (let a in E){
		for (let b in E){
		    if (a >= b) continue
		    if (E[a][1] == E[b][0] ){
			let min = Math.min(E[a][0],E[b][1])
			let max = Math.max(E[a][0],E[b][1])
			for (let c in E){
			    if ((E[c][0]==min && E[c][1]==max) || (E[c][1]==min && E[c][0]==max)){
				F.push([a,b,c]);
			    }
			}
		    }
		    if (E[a][0] == E[b][1] ){
			let min = Math.min(E[a][1],E[b][0])
			let max = Math.max(E[a][1],E[b][0])
			for (let c in E){
			    if ((E[c][0]==min && E[c][1]==max) || (E[c][1]==min && E[c][0]==max)){
				F.push([a,b,c]);
			    }
			}
		    }
		}
	    }
	    console.log(F)
	    this.V = V;
	    this.E = E;
	    this.F = F;
	},
  },


    components: {
	face,
	edge,
	vertex
    }
}
</script>
