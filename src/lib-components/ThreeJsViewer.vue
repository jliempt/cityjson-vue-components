<template>
  <div
    id="viewer"
    class="col-12 px-0 h-100"
  />
</template>

<script>
import $ from 'jquery';
import * as THREE from 'three';
// import { GeometryCompressionUtils } from 'three/examples/jsm/utils/GeometryCompressionUtils.js';
import OrbitControls from 'three-orbitcontrols';
import earcut from 'earcut';
import JSONStream from 'JSONStream';
import request from 'request';
import Stream from 'event-stream';
import ReadableStreamClone from 'readable-stream-clone';

export default {
	name: 'ThreeJsViewer',
	props: {
		citymodel: Object,
		selected_objid: String,
		object_colors: {
			type: Object,
			default: function () {

				return {
					"Building": 0x7497df,
					"BuildingPart": 0x7497df,
					"BuildingInstallation": 0x7497df,
					"Bridge": 0x999999,
					"BridgePart": 0x999999,
					"BridgeInstallation": 0x999999,
					"BridgeConstructionElement": 0x999999,
					"CityObjectGroup": 0xffffb3,
					"CityFurniture": 0xcc0000,
					"GenericCityObject": 0xcc0000,
					"LandUse": 0xffffb3,
					"PlantCover": 0x39ac39,
					"Railway": 0x000000,
					"Road": 0x999999,
					"SolitaryVegetationObject": 0x39ac39,
					"TINRelief": 0xffdb99,
					"TransportSquare": 0x999999,
					"Tunnel": 0x999999,
					"TunnelPart": 0x999999,
					"TunnelInstallation": 0x999999,
					"WaterBody": 0x4da6ff
				};

			}
		},

		background_color: {
			type: Number,
			default: 0xd9eefc
		}

	},
	data() {

		return {
			camera_init: false
		};

	},

	watch: {

		background_color: function ( newVal, ) {

			this.renderer.setClearColor( newVal );

			this.renderer.render( this.scene, this.camera );

		},

		object_colors: {
			handler: function ( newVal, ) {

				for ( var i = 0; i < this.meshes.length; i ++ )
					this.meshes[ i ].material.color.setHex( newVal[ this.citymodel.CityObjects[ this.meshes[ i ].name ].type ] );

				this.renderer.render( this.scene, this.camera );

			},
			deep: true

		},

		selected_objid: function ( newID, oldID ) {

			if ( oldID != null && oldID in this.citymodel.CityObjects ) {

				var coType = this.citymodel.CityObjects[ oldID ].type;
				var color = new THREE.Color( this.object_colors[ coType ] );

				this.updateCOColor( color, oldID );

			}

			if ( newID != null ) {

				color = new THREE.Color( 0xdda500 );

				this.updateCOColor( color, newID );

			}

			this.renderer.render( this.scene, this.camera );

		}

	},
	async beforeCreate() {

		this.scene = null;
		this.camera = null;
		this.renderer = null;
		this.controls = null;
		this.raycaster = null;
		this.mouse = null;
		this.geometry = new THREE.BufferGeometry();
		this.mesh = null;
		this.mesh_index = {};
		this.faceIDs = [];
		this.idFaces = {};
		this.vertices = [];
		this.colors = [];
		this.indices = [];
		this.cmvertices = [];
		this.cos = {};
		this.i = 0;

	},

	async mounted() {

		this.$emit( 'rendering', true );

		setTimeout( async () => {

			var self = this;

			this.initScene();

			const cmURL = 'http://localhost:8080/denhaag.json';

			// Retrieve citymodel and clone twice. First clone is used to retrieve vertices from buffer, second to iteratively retrieve CityObjects.
			var rawStream = request( { url: cmURL } );

			var streamClone1 = new ReadableStreamClone( rawStream );
			var streamClone2 = new ReadableStreamClone( rawStream );

			// Already render before streaming has finished, so that the background is shown in the meantime.
			this.renderer.render( this.scene, this.camera );

			await new Promise( resolve =>
				streamClone1
					.pipe( JSONStream.parse( 'vertices' ) )
					.pipe( Stream.mapSync( function ( data ) {

						// Store vertices
						self.cmvertices = data;

					} ) )
					.on( 'end', resolve ) );

			console.log( "Vertices done" );

			await new Promise( resolve =>
				streamClone2
					.pipe( JSONStream.parse( 'CityObjects.$*' ) )
					.pipe( Stream.mapSync( function ( data ) {

						// Iteratively parse CityObjects
						self.parseObject( data.key, data.value );

					} ) )
					.on( 'end', resolve ) );

			console.log( "COs done" );

			// Parse it all into a BufferGeometry and add to scene
			self.createGeometry();
			self.renderer.render( self.scene, self.camera );

			$( "#viewer" ).dblclick( function ( eventData ) {

				if ( eventData.button == 0 ) { //leftClick

					self.handleClick();

				}

			} );

			this.$emit( 'rendering', false );

		}, 25 );

	},

	methods: {

		updateCOColor( color, coID ) {

			var firstFaceID = this.idFaces[ coID ][ 0 ];
			var lastFaceID = this.idFaces[ coID ][ 1 ];

			for ( var i = firstFaceID; i <= lastFaceID; i ++ ) {

				var vertices = [];

				vertices.push( this.mesh.geometry.index.array[ i * 3 ] );
				vertices.push( this.mesh.geometry.index.array[ i * 3 + 1 ] );
				vertices.push( this.mesh.geometry.index.array[ i * 3 + 2 ] );

				for ( var v of vertices ) {

					this.mesh.geometry.attributes.color.array[ v * 3 ] = color.r;
					this.mesh.geometry.attributes.color.array[ v * 3 + 1 ] = color.g;
					this.mesh.geometry.attributes.color.array[ v * 3 + 2 ] = color.b;

				}

			}

			this.mesh.geometry.colorsNeedUpdate = true;
			this.mesh.geometry.attributes.color.needsUpdate = true;

		},

		handleClick() {

			var rect = this.renderer.domElement.getBoundingClientRect();
			this.mouse.x = ( ( event.clientX - rect.left ) / this.renderer.domElement.clientWidth ) * 2 - 1;
			this.mouse.y = - ( ( event.clientY - rect.top ) / this.renderer.domElement.clientHeight ) * 2 + 1;

			this.raycaster.setFromCamera( this.mouse, this.camera );
			var intersects = this.raycaster.intersectObject( this.mesh );

			if ( intersects.length == 0 ) {

				this.$emit( 'object_clicked', null );
				return;

			}

			var cityObjId = this.faceIDs[ intersects[ 0 ].faceIndex ];
			this.$emit( 'object_clicked', cityObjId );

		},

		initScene() {

			this.scene = new THREE.Scene();
			var ratio = $( "#viewer" ).width() / $( "#viewer" ).height();
			this.camera = new THREE.PerspectiveCamera( 60, ratio, 0.001, 1000 );
			this.camera.up.set( 0, 0, 1 );
			this.camera.position.set( 0, 0, 2 );
			this.camera.lookAt( 0, 0, 0 );

			this.renderer = new THREE.WebGLRenderer( { antialias: true } );
			var viewer = document.getElementById( "viewer" );
			viewer.appendChild( this.renderer.domElement );
			this.renderer.setSize( $( "#viewer" ).width(), $( "#viewer" ).height() );
			this.renderer.setClearColor( this.background_color );
			this.renderer.shadowMap.enabled = true;
			this.renderer.shadowMap.type = THREE.PCFSoftShadowMap;

			this.raycaster = new THREE.Raycaster();
			this.mouse = new THREE.Vector2();

			var ambLight = new THREE.AmbientLight( 0xFFFFFF, 0.7 );
			ambLight.name = "ambLight";
			var spotLight = new THREE.SpotLight( 0xDDDDDD, 0.4 );
			spotLight.name = "spotLight";
			spotLight.position.set( 0, - 1, 1 );
			spotLight.target = this.scene;
			spotLight.castShadow = true;
			this.scene.add( spotLight, ambLight );

			let self = this;
			this.controls = new OrbitControls( this.camera, this.renderer.domElement );
			this.controls.addEventListener( 'change', function () {

				self.renderer.render( self.scene, self.camera );

			} );

			this.controls.screenSpacePanning = true;

		},

		clearScene() {

			for ( var i = this.scene.children.length - 1; i >= 0; i -- ) {

				if ( this.scene.children[ i ].name != "ambLight" && this.scene.children[ i ].name != "spotLight" ) {

					this.scene.remove( this.scene.children[ i ] );

				}

			}

			// TODO: properly reinitialise all properties and test if this function works well.
			this.mesh = null;
			this.geometry = new THREE.BufferGeometry();
			this.faceIDs = [];
			this.idFaces = {};
			this.vertices = [];
			this.colors = [];
			this.indices = [];

		},

		async createGeometry() {

			var material = new THREE.MeshLambertMaterial();
			material.vertexColors = true;

			this.geometry.setIndex( this.indices );
			this.geometry.setAttribute( 'position', new THREE.Float32BufferAttribute( this.vertices, 3 ) );
			this.geometry.setAttribute( 'color', new THREE.Float32BufferAttribute( this.colors, 3 ) );
			this.geometry.computeBoundingSphere();

			const center = this.geometry.boundingSphere.center;
			const radius = this.geometry.boundingSphere.radius;

			const s = radius === 0 ? 1 : 1.0 / radius;

			const matrix = new THREE.Matrix4();
			matrix.set(
				s, 0, 0, - s * center.x,
				0, s, 0, - s * center.y,
				0, 0, s, - s * center.z,
				0, 0, 0, 1
			);

			this.geometry.applyMatrix4( matrix );

			this.geometry.computeVertexNormals();

			console.log( this.geometry );

			this.mesh = new THREE.Mesh( this.geometry, material );
			this.mesh.castShadow = true;
			this.mesh.receiveShadow = true;

			this.scene.add( this.mesh );

			delete this.cmvertices;

		},

		async parseObject( id, cityObj ) {

			if ( this.i % 100 == 0 ) {

				console.log( this.i );

			}

			this.i += 1;

			const coType = cityObj.type;
			const color = new THREE.Color( this.object_colors[ coType ] );
			const firstFaceID = this.faceIDs.length;

			var vertices = [];

			if ( ! ( cityObj.geometry &&
        cityObj.geometry.length > 0 ) ) {

				return;

			}

			for ( var geom_i = 0; geom_i < cityObj.geometry.length; geom_i ++ ) {

				//each geometrytype must be handled different
				var geomType = cityObj.geometry[ geom_i ].type;

				var i, j;

				if ( geomType == "Solid" ) {

					var shells = cityObj.geometry[ geom_i ].boundaries;

					for ( i = 0; i < shells.length; i ++ ) {

						await this.parseShell( shells[ i ], vertices );

					}

				} else if ( geomType == "MultiSurface" || geomType == "CompositeSurface" ) {

					var surfaces = cityObj.geometry[ geom_i ].boundaries;

					await this.parseShell( surfaces, vertices );

				} else if ( geomType == "MultiSolid" || geomType == "CompositeSolid" ) {

					var solids = cityObj.geometry[ geom_i ].boundaries;

					for ( i = 0; i < solids.length; i ++ ) {

						for ( j = 0; j < solids[ i ].length; j ++ ) {

							await this.parseShell( solids[ i ][ j ], vertices );

						}

					}

				}

			}

			var uniqueVertices = [ ...new Set( vertices ) ];
			const length = this.vertices.length / 3;

			for ( v of uniqueVertices ) {

				this.vertices.push( this.cmvertices[ v ][ 0 ] );
				this.vertices.push( this.cmvertices[ v ][ 1 ] );
				this.vertices.push( this.cmvertices[ v ][ 2 ] );

				this.colors.push( color.r, color.g, color.b );

			}

			for ( var v = 0; v < vertices.length; v += 3 ) {

				var i0 = uniqueVertices.indexOf( vertices[ v ] ) + length;
				var i1 = uniqueVertices.indexOf( vertices[ v + 1 ] ) + length;
				var i2 = uniqueVertices.indexOf( vertices[ v + 2 ] ) + length;

				this.indices.push( i0, i1, i2 );

				this.faceIDs.push( id );

			}

			this.idFaces[ id ] = [ firstFaceID, this.faceIDs.length - 1 ];

			return ( "" );

		},

		async parseShell( boundaries, vertices ) {

			var i, j;

			for ( i = 0; i < boundaries.length; i ++ ) {

				var boundary = [];
				var holes = [];

				for ( j = 0; j < boundaries[ i ].length; j ++ ) {

					if ( boundary.length > 0 ) {

						holes.push( boundary.length );

					}

					boundary.push( ...boundaries[ i ][ j ] );

				}


				if ( boundary.length == 3 ) {

					vertices.push( boundary[ 0 ],
						boundary[ 1 ],
						boundary[ 2 ] );

				} else if ( boundary.length > 3 ) {

					//create list of points
					var pList = [];
					var k;
					for ( k = 0; k < boundary.length; k ++ ) {

						pList.push( {
							x: this.cmvertices[ boundary[ k ] ][ 0 ],
							y: this.cmvertices[ boundary[ k ] ][ 1 ],
							z: this.cmvertices[ boundary[ k ] ][ 2 ]
						} );

					}

					//get normal of these points
					var normal = await this.get_normal_newell( pList );

					//convert to 2d (for triangulation)
					var pv = [];
					for ( k = 0; k < pList.length; k ++ ) {

						var re = await this.to_2d( pList[ k ], normal );
						pv.push( re.x );
						pv.push( re.y );

					}

					//triangulate
					var tr = await earcut( pv, holes, 2 );

					//create faces based on triangulation
					for ( k = 0; k < tr.length; k += 3 ) {

						for ( var n = 0; n < 3; n ++ ) {

							vertices.push( boundary[ tr[ k + n ] ] );

						}

					}

				}

			}

		},

		//-- calculate normal of a set of points
		get_normal_newell( indices ) {

			// find normal with Newell's method
			var n = [ 0.0, 0.0, 0.0 ];

			for ( var i = 0; i < indices.length; i ++ ) {

				var nex = i + 1;

				if ( nex == indices.length ) {

					nex = 0;

				}

				n[ 0 ] = n[ 0 ] + ( ( indices[ i ].y - indices[ nex ].y ) * ( indices[ i ].z + indices[ nex ].z ) );
				n[ 1 ] = n[ 1 ] + ( ( indices[ i ].z - indices[ nex ].z ) * ( indices[ i ].x + indices[ nex ].x ) );
				n[ 2 ] = n[ 2 ] + ( ( indices[ i ].x - indices[ nex ].x ) * ( indices[ i ].y + indices[ nex ].y ) );

			}

			var b = new THREE.Vector3( n[ 0 ], n[ 1 ], n[ 2 ] );
			return ( b.normalize() );

		},

		to_2d( p, n ) {

			p = new THREE.Vector3( p.x, p.y, p.z );
			var x3 = new THREE.Vector3( 1.1, 1.1, 1.1 );
			if ( x3.distanceTo( n ) < 0.01 ) {

				x3.add( new THREE.Vector3( 1.0, 2.0, 3.0 ) );

			}

			var tmp = x3.dot( n );
			var tmp2 = n.clone();
			tmp2.multiplyScalar( tmp );
			x3.sub( tmp2 );
			x3.normalize();
			var y3 = n.clone();
			y3.cross( x3 );
			let x = p.dot( x3 );
			let y = p.dot( y3 );
			var re = { x: x, y: y };
			return re;

		}
	}
};
</script>
