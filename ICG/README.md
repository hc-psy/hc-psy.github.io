---
layout: plain
sitemap: false
---

# ICG Project

* this list will be replaced by the toc
{:toc .large-only}

<head>
<meta http-equiv="content-type" content="text/html; charset=ISO-8859-1">

<script type="text/javascript" src="./js/glMatrix-0.9.5.min.js"></script>
<script type="text/javascript" src="./js/webgl-utils.js"></script>
<script type="text/javascript" src="./js/icg.js"></script>
<!-- <link rel="stylesheet" href="https://maxcdn.bootstrapcdn.com/bootstrap/4.5.2/css/bootstrap.min.css">
<script src="https://ajax.googleapis.com/ajax/libs/jquery/3.5.1/jquery.min.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/popper.js/1.16.0/umd/popper.min.js"></script>
<script src="https://maxcdn.bootstrapcdn.com/bootstrap/4.5.2/js/bootstrap.min.js"></script> -->
<!-- <link href="./styles/style.css" rel="stylesheet" type="text/css"> -->
<link href="https://fonts.googleapis.com/icon?family=Material+Icons" rel="stylesheet">

<!-- multiple shaders -->
<script id="flatf" type="fragment">
    #extension GL_OES_standard_derivatives : enable
    precision mediump float;

    uniform vec3 uLightPosition[3];
    uniform vec3 uLightColor[3];
    uniform float uLightEnable[3];

    uniform float uLightShine[3];
    uniform float uLightKa[3];
    uniform float uLightKd[3];
    uniform float uLightKs[3];
    
    varying vec3 mvVertex;
    varying vec4 fragcolor;
    varying vec3 vLightDirection[3];
    
    
    void main(void) {
        vec3 X = dFdx(mvVertex);
        vec3 Y = dFdy(mvVertex);
        vec3 normal = normalize(cross(X,Y));
    
        vec3 V = -normalize(mvVertex);
        vec3 light = vec3(0.0, 0.0, 0.0);
    
        for (int i = 0; i < 3; i++) {
            if(uLightEnable[i] == 0.0) {
                continue;
            }
            vec3 R = reflect(-vLightDirection[i], normal);
            
            float specularCos = pow(max(dot(R, V), 0.0), uLightShine[i]) * uLightKs[i];
            float diffuseCos = max(dot(normal, vLightDirection[i]), 0.0) * uLightKd[i];
    
            light += uLightColor[i] * uLightKa[i] + uLightColor[i] * diffuseCos + uLightColor[i] * specularCos;
        }
    
        // vec3 lightDirection = normalize(uLightPosition[0] - mvVertex);
    
        // float light = max(0.0, dot(lightDirection, normal));
    
        gl_FragColor = vec4(fragcolor.rgb * light, 1.0);
    }
</script>

<script id="flatv" type="vertex">
    precision mediump float;

    attribute vec3 aVertexPosition;
    attribute vec3 aFrontColor;
    attribute vec3 aVertexNormal;

    uniform mat4 uMVMatrix;
    uniform mat4 uPMatrix;
    uniform mat4 uNVMatrix;

    uniform vec3 uLightPosition[3];
    uniform vec3 uLightColor[3];
    uniform float uLightEnable[3];


    varying vec3 mvVertex;
    varying vec4 fragcolor;
    varying vec3 vLightDirection[3];

    attribute vec2 aTextureCoord; // for texture
    uniform int uTexture; // for texture
    uniform sampler2D uSampler; // for texture

    void main(void) {
        mvVertex = (uMVMatrix * vec4(aVertexPosition, 1.0)).xyz;

        aVertexNormal; // activate for not warning

        for (int i = 0; i < 3; i++) {
            vLightDirection[i] = normalize(uLightPosition[i]-mvVertex);
        }

        fragcolor = vec4(aFrontColor, 1.0);
        
        if (uTexture != 0){
            fragcolor = texture2D(uSampler, vec2(aTextureCoord.s, aTextureCoord.t));
        } // for texture block
        
        gl_Position = uPMatrix * uMVMatrix * vec4(aVertexPosition, 1.0);
    }
</script>

<script id="gouraudf" type="fragment">
    precision mediump float;

    varying vec4 fragcolor;

    void main(void) {
        gl_FragColor = fragcolor;
    }
</script>

<script id="gouraudv" type="vertex">
    attribute vec3 aVertexPosition;
    attribute vec3 aFrontColor;
    attribute vec3 aVertexNormal;

    uniform mat4 uMVMatrix;
    uniform mat4 uPMatrix;
    uniform mat4 uNVMatrix;

    uniform vec3 uLightPosition[3];
    uniform vec3 uLightColor[3];
    uniform float uLightEnable[3];

    uniform float uLightShine[3];
    uniform float uLightKa[3];
    uniform float uLightKd[3];
    uniform float uLightKs[3];

    varying vec4 fragcolor;

    attribute vec2 aTextureCoord; // for texture
    uniform int uTexture; // for texture
    uniform sampler2D uSampler; // for texture

    void main(void) {
        vec3 mvVertex = (uMVMatrix * vec4(aVertexPosition, 1.0)).xyz;

        // V, N, L, H
        vec3 V = -normalize(mvVertex);
        vec3 N = normalize(mat3(uMVMatrix) * aVertexNormal);

        vec3 light = vec3(0.0, 0.0, 0.0);

        vec3 useColor;
        vec4 textureColor; //

        useColor = aFrontColor;
        
        if (uTexture != 0){
            textureColor = texture2D(uSampler, vec2(aTextureCoord.s, aTextureCoord.t));
            useColor = vec3(textureColor.rgb);
        } //

        for (int i = 0; i<3; i++) {
            if(uLightEnable[i] == 0.0) {
                continue;
            }

            vec3 L = normalize(uLightPosition[i] - mvVertex);
            vec3 H = normalize(V + L);

            // gouraud = ambient + diffuse + specular
            vec3 gouraudShading = vec3(0.0);

            vec3 ambient = uLightColor[i] * uLightKa[i] * useColor;

            float cosTheta = max(dot(L, N), 0.0);
            vec3 diffuse = uLightColor[i] * uLightKd[i] * useColor * cosTheta;

            float cosAlpha = max(dot(H, N), 0.0);
            vec3 specular = uLightColor[i] * uLightKs[i] * pow(cosAlpha, uLightShine[i]); //?useColor
            if(dot(L, N) < 0.0){
                specular = vec3(0.0);
            }

            gouraudShading = ambient + diffuse + specular;
            light += gouraudShading;
        }

        if (uTexture != 0){
            fragcolor = vec4(light, textureColor.a);
        } else {
            fragcolor = vec4(light, 1.0);
        } //

        gl_Position = uPMatrix * uMVMatrix * vec4(aVertexPosition, 1.0);
    }
</script>

<script id="phongf" type="fragment">
    precision mediump float;

    uniform vec3 uLightPosition[3];
    uniform vec3 uLightColor[3];
    uniform float uLightEnable[3];
    
    uniform float uLightShine[3];
    uniform float uLightKa[3];
    uniform float uLightKd[3];
    uniform float uLightKs[3];

    varying vec3 mvVertex;
    varying vec4 fragcolor;
    varying vec3 normalInterp;


    uniform sampler2D uSampler;
    varying vec2 vTextureCoord;
    uniform highp int uTexture; // for texture

    void main(void) {
        vec3 N = normalize(normalInterp);

        vec3 light = vec3(0.0, 0.0, 0.0);

        for (int i=0; i<3; i++) {
            if(uLightEnable[i] == 0.0) {
                    continue;
            }

            vec3 L = normalize(uLightPosition[i] - mvVertex);

            float lambertian = max(dot(N, L), 0.0);
            float specularCos = 0.0;

            if(lambertian > 0.0) {
                vec3 R = reflect(-L, N);       // Reflected light vector
                vec3 V = normalize(-mvVertex); // Vector to viewer

                float specAngle = max(dot(R, V), 0.0);
                specularCos = pow(specAngle, uLightShine[i]);
            }

            vec3 ambient = uLightKa[i] * uLightColor[i];
            vec3 diffuse = uLightKd[i] * lambertian * uLightColor[i];
            vec3 specular = uLightKs[i] * specularCos * uLightColor[i];

            light += ambient + diffuse + specular;
        }

        if (uTexture != 0){
            vec4 fragmentColor;
            fragmentColor = texture2D(uSampler, vec2(vTextureCoord.s, vTextureCoord.t));
            gl_FragColor = vec4(fragmentColor.rgb * light, fragmentColor.a);
        } else {
            gl_FragColor = vec4(fragcolor.rgb * light, 1.0);
        }//
    }
    
</script>

<script id="phongv" type="vertex">
    precision mediump float;

    attribute vec3 aVertexPosition;
    attribute vec3 aFrontColor;
    attribute vec3 aVertexNormal;

    uniform mat4 uMVMatrix;
    uniform mat4 uPMatrix;
    uniform mat4 uNVMatrix;

    varying vec3 mvVertex;
    varying vec4 fragcolor;
    varying vec3 normalInterp;

    attribute vec2 aTextureCoord; //
    varying vec2 vTextureCoord; //
    uniform int uTexture;

    void main(void) {
        mvVertex = (uMVMatrix * vec4(aVertexPosition, 1.0)).xyz;
        normalInterp = normalize(mat3(uNVMatrix) * aVertexNormal);
        fragcolor = vec4(aFrontColor, 1.0);
        if (uTexture != 0){
            vTextureCoord = aTextureCoord; //
        }
        gl_Position = uPMatrix * uMVMatrix * vec4(aVertexPosition, 1.0);
    }
</script>

<script id="celf" type="fragment">
    precision mediump float;

    uniform mat4 uMVMatrix;
    uniform mat4 uPMatrix;
    uniform mat4 uNVMatrix;
    uniform vec3 uLightPosition[3];
    uniform vec3 uLightColor[3];
    uniform float uLightEnable[3];
    uniform float uLightShine[3];
    uniform float uLightKa[3];
    uniform float uLightKd[3];
    uniform float uLightKs[3];

    varying vec3 mvVertex;
    varying vec3 normal;
    varying vec4 fragcolor;


    void main(void) {

        float intensity = 0.0;
        vec3 allLightColor;
        for (int i=0; i<3; i++) {
            if(uLightEnable[i] == 0.0) {
                continue;
            }
            vec3 lightDir = normalize(uLightPosition[i] - mvVertex);
            intensity += dot(lightDir, normalize(mat3(uNVMatrix) * normal));
            allLightColor += uLightColor[i];

            uLightShine[i];
            uLightKa[i];
            uLightKd[i];
            uLightKs[i]; 
        }
         
        float level = 1.0;
        if (intensity > 0.98)
            level = 1.0;
        else if (intensity > 0.9)
            level = 0.62;
        else if (intensity > 0.75)
            level = 0.54;
        else if (intensity > 0.55)
            level = 0.41;
        else if (intensity > 0.25)
            level = 0.27;
        else
            level = 0.1;
        
        gl_FragColor = vec4(fragcolor.rgb * allLightColor * level, fragcolor.a);
    }
    
</script>

<script id="celv" type="vertex">
    precision mediump float;

    attribute vec3 aVertexPosition;
    attribute vec3 aFrontColor;
    attribute vec3 aVertexNormal;
    
    uniform mat4 uMVMatrix;
    uniform mat4 uPMatrix;
    uniform vec3 uLightPosition[3];
    uniform vec3 uLightColor[3];
    uniform float uLightEnable[3];
    
    varying vec3 mvVertex;
    varying vec3 normal;
    varying vec4 fragcolor;

    attribute vec2 aTextureCoord; //
    uniform int uTexture; //
    uniform sampler2D uSampler; // for texture

    void main(void) {
        mvVertex = (uMVMatrix * vec4(aVertexPosition, 1.0)).xyz;
        normal = normalize(aVertexNormal);
        fragcolor = vec4(aFrontColor, 1.0);
        if (uTexture != 0){
            fragcolor = texture2D(uSampler, vec2(aTextureCoord.s, aTextureCoord.t));
        }
        gl_Position = uPMatrix * uMVMatrix * vec4(aVertexPosition, 1.0);
    }
</script>

</head>

<style>
    h2 {
        text-align: center;
    }
</style>

<body onload="webGLStart();">
<canvas id="ICG-canvas" width="1024" height="760" style="width: 100%;"></canvas>
</body>

## Basic Settings

<div class="adbox">
    <form>
        <div class="form-row">
            <label class="col-3 col-form-label text-center">Model</label>
            <div class="form-group col-3">
                <select class="form-control" id="model0" onchange="setModel(value, 0);">
                    <option value="Teapot">Teapot</option>
                    <option value="Csie">CSIE</option>
                    <option value="Car_road">Car</option>
                    <option value="Church_s">Church</option>
                    <option value="Easter">Easter</option>
                    <option value="Fighter">Fighter</option>
                    <option value="Kangaroo">Kangaroo</option>
                    <option value="Longteap">Long Teapot</option>
                    <option value="Mercedes">Mercedes</option>
                    <option value="Mig27">Mig27</option>
                    <option value="Tomcat">Tomcat</option>
                    <option value="Patchair">Pat Chair</option>
                    <option value="Plant">Plant</option>
                </select>
                <small class="form-text text-muted">
                    Select your first model.
                </small>
            </div>
            <div class="form-group col-3">
                <select class="form-control" id="model1" onchange="setModel(value, 1);">
                    <option value="Teapot">Teapot</option>
                    <option value="Csie">CSIE</option>
                    <option value="Car_road">Car</option>
                    <option value="Church_s">Church</option>
                    <option value="Easter">Easter</option>
                    <option value="Fighter">Fighter</option>
                    <option value="Kangaroo">Kangaroo</option>
                    <option value="Longteap">Long Teapot</option>
                    <option value="Mercedes">Mercedes</option>
                    <option value="Mig27">Mig27</option>
                    <option value="Tomcat">Tomcat</option>
                    <option value="Patchair">Pat Chair</option>
                    <option value="Plant">Plant</option>
                </select>
                <small class="form-text text-muted">
                    Select your second model.
                </small>
            </div>
            <div class="form-group col-3">
                <select class="form-control" id="model2" onchange="setModel(value, 2);">
                    <option value="Teapot">Teapot</option>
                    <option value="Csie">CSIE</option>
                    <option value="Car_road">Car</option>
                    <option value="Church_s">Church</option>
                    <option value="Easter">Easter</option>
                    <option value="Fighter">Fighter</option>
                    <option value="Kangaroo">Kangaroo</option>
                    <option value="Longteap">Long Teapot</option>
                    <option value="Mercedes">Mercedes</option>
                    <option value="Mig27">Mig27</option>
                    <option value="Tomcat">Tomcat</option>
                    <option value="Patchair">Pat Chair</option>
                    <option value="Plant">Plant</option>
                </select>
                <small class="form-text text-muted">
                    Select your third model.
                </small>
            </div>
        </div>
        <div class="form-row">
            <label class="col-3 col-form-label text-center">Shader</label>
            <div class="form-group col-3">
                <select class="form-control" onchange="setShader(value, 0);">
                    <option value="0"  selected>Flat Shading</option>
                    <option value="1">Gouraud Shading</option>
                    <option value="2">Phong Shading</option>
                    <option value="3">Cel Shading</option>
                </select>
                <small class="form-text text-muted">
                    Select your shader of the first model.
                </small>
            </div>
            <div class="form-group col-3">
                <select class="form-control" onchange="setShader(value, 1);">
                    <option value="0">Flat Shading</option>
                    <option value="1"  selected>Gouraud Shading</option>
                    <option value="2">Phong Shading</option>
                    <option value="3">Cel Shading</option>
                </select>
                <small class="form-text text-muted">
                    Select your shader of the second model.
                </small>
            </div>
            <div class="form-group col-3">
                <select class="form-control" onchange="setShader(value, 2);">
                    <option value="0">Flat Shading</option>
                    <option value="1">Gouraud Shading</option>
                    <option value="2"  selected>Phong Shading</option>
                    <option value="3">Cel Shading</option>
                </select>
                <small class="form-text text-muted">
                    Select your shader of the third model.
                </small>
            </div>
        </div>
        <div class="form-row">
            <label class="col-3 col-form-label text-center">Draw Mode</label>
            <div class="form-group col-3">
                <select class="form-control" id="drawmode0">
                    <option value="0">Points</option>
                    <option value="1">Line Stripe</option>
                    <option value="2">Line Loop</option>
                    <option value="3">Lines</option>
                    <option value="4" selected>Triangles Stripe</option>
                    <option value="5">Triangles Fan</option>
                    <option value="6">Triangles</option>
                </select>
                <small class="form-text text-muted">
                    Select your draw mode of the first model.
                </small>
            </div>
            <div class="form-group col-3">
                <select class="form-control" id="drawmode1">
                    <option value="0">Points</option>
                    <option value="1">Line Stripe</option>
                    <option value="2">Line Loop</option>
                    <option value="3">Lines</option>
                    <option value="4" selected>Triangles Stripe</option>
                    <option value="5">Triangles Fan</option>
                    <option value="6">Triangles</option>
                </select>
                <small class="form-text text-muted">
                    Select your draw mode of the second model.
                </small>
            </div>
            <div class="form-group col-3">
                <select class="form-control" id="drawmode2">
                    <option value="0">Points</option>
                    <option value="1">Line Stripe</option>
                    <option value="2">Line Loop</option>
                    <option value="3">Lines</option>
                    <option value="4" selected>Triangles Stripe</option>
                    <option value="5">Triangles Fan</option>
                    <option value="6">Triangles</option>
                </select>
                <small class="form-text text-muted">
                    Select your draw mode of the third model.
                </small>
            </div>
        </div>
        <div class="form-row">
            <label class="col-3 col-form-label text-center">Texture (only for teapot)</label>
            <div class="form-group col-3">
                <select class="form-control" id="texture0">
                    <option value="0">Original Texture</option>
                    <option value="1">Louis Vuitton</option>
                    <option value="2">Burberry</option>
                    <option value="3">Bottega Veneta</option>
                    <option value="4">Gucci</option>
                    <option value="5">Hermes</option>
                </select>
                <small class="form-text text-muted">
                    Select your texture of the first teapot model.
                </small>
            </div>
            <div class="form-group col-3">
                <select class="form-control" id="texture1">
                    <option value="0">Original Texture</option>
                    <option value="1">Louis Vuitton</option>
                    <option value="2">Burberry</option>
                    <option value="3">Bottega Veneta</option>
                    <option value="4">Gucci</option>
                    <option value="5">Hermes</option>
                </select>
                <small class="form-text text-muted">
                    Select your texture of the second teapot model.
                </small>
            </div>
            <div class="form-group col-3">
                <select class="form-control" id="texture2">
                    <option value="0">Original Texture</option>
                    <option value="1">Louis Vuitton</option>
                    <option value="2">Burberry</option>
                    <option value="3">Bottega Veneta</option>
                    <option value="4">Gucci</option>
                    <option value="5">Hermes</option>
                </select>
                <small class="form-text text-muted">
                    Select your texture of the third teapot model.
                </small>
            </div>
        </div>
    </form>
</div>

## Transformation Settings

<div class="adbox">
    <form>
        <div class="form-row align-items-center">
            <label class="col-3 col-form-label text-center">Scaling</label>
            <div class="form-group col-3">
                <input type="range" id="scale-value" class="custom-range" value="100" min="1" max="200" oninput="svt.value = this.value/100">
                <small class="form-text text-secondary">
                    Scale is <output name="svt">1</output>
                </small>
            </div>
            <label class="col-3 col-form-label text-center ">
                <button class="btn" type="button" onclick="randomshear()">
                    Shear
                </button>    
            </label>
            <div class="form-group col-3">
                <div class="form-row h-33 align-items-center">
                    <div class="form-group col-4">
                        <input type="number" class="form-control" id="m11" value="1" placeholder="1" style="-moz-appearance:textfield;">
                    </div>
                    <div class="form-group col-4">
                        <input type="number" class="form-control" id="m12" value="0" placeholder="0" style="-moz-appearance:textfield;">
                    </div>
                    <div class="form-group col-4">
                        <input type="number" class="form-control" id="m13" value="0" placeholder="0" style="-moz-appearance:textfield;">
                    </div>
                </div>
                <div class="form-row h-33 align-items-center">
                    <div class="form-group col-4">
                        <input type="number" class="form-control" id="m21" value="0" placeholder="0" style="-moz-appearance:textfield;">
                    </div>
                    <div class="form-group col-4">
                        <input type="number" class="form-control" id="m22" value="1" placeholder="1" style="-moz-appearance:textfield;">
                    </div>
                    <div class="form-group col-4">
                        <input type="number" class="form-control" id="m23" value="0" placeholder="0" style="-moz-appearance:textfield;">
                    </div>
                </div>
                <div class="form-row h-33 align-items-center">
                    <div class="form-group col-4">
                        <input type="number" class="form-control" id="m31" value="0" placeholder="0" style="-moz-appearance:textfield;">
                    </div>
                    <div class="form-group col-4">
                        <input type="number" class="form-control" id="m32" value="0" placeholder="0" style="-moz-appearance:textfield;">
                    </div>
                    <div class="form-group col-4">
                        <input type="number" class="form-control" id="m33" value="1" placeholder="1" style="-moz-appearance:textfield;">
                    </div>
                </div>
            </div>
        </div>
        <div class="form-row">
            <label class="col-3 col-form-label text-center">Rotation</label>
            <div class="form-group col-3">
                <input type="range" id="rxin" class="custom-range" value="100" min="-100" max="100" oninput="rx.value = this.value/100">
                <small class="form-text text-secondary">
                    X-axis vector is [<output name="rx">1</output>]
                </small>
            </div>
            <div class="form-group col-3">
                <input type="range" id="ryin" class="custom-range" value="100" min="-100" max="100" oninput="ry.value = this.value/100">
                <small class="form-text text-secondary">
                    Y-axis vector is [<output name="ry">1</output>]
                </small>
            </div>
            <div class="form-group col-3">
                <input type="range" id="rzin" class="custom-range" value="100" min="-100" max="100" oninput="rz.value = this.value/100">
                <small class="form-text text-secondary">
                    Z-axis vector is [<output name="rz">1</output>]
                </small>
            </div>
        </div>
        <div class="form-row">
            <label class="col-3 col-form-label text-center">Translation</label>
            <div class="form-group col-3">
                <input type="range" id="tin0" class="custom-range" value="100" min="0" max="100" oninput="tx.value = this.value/100">
                <small class="form-text text-secondary">
                    X-axis translation range is <output name="tx">1</output>
                </small>
            </div>
            <div class="form-group col-3">
                <input type="range" id="tin1" class="custom-range" value="100" min="0" max="100" oninput="ty.value = this.value/100">
                <small class="form-text text-secondary">
                    Y-axis translation range is <output name="ty">1</output>
                </small>
            </div>
            <div class="form-group col-3">
                <input type="range" id="tin2" class="custom-range" value="100" min="0" max="100" oninput="tz.value = this.value/100">
                <small class="form-text text-secondary">
                    Z-axis translation range is <output name="tz">1</output>
                </small>
            </div>
        </div>
        <div class="form-row">
            <label class="col-3 text-center">
                Speed 
            </label>
            <div class="col-1 text-center">
            <div id="speedspinner" class="spinner-border" style="animation-delay: 0ms; animation-duration: 0.2s;"></div>
            </div>
            <div class="form-group col-8">
                <input type="range" id="speedin" class="custom-range" value="500" min="0" max="1000" oninput="elapseCalculation(this.value)">
            </div>
        </div>
    </form>
</div>

## Light Settings

<div class="adbox">
    <div id="accordionlight">
        <div class="card">
            <div class="card-header" id="lightCard1">
            <h5 class="mb-0 inline">
                <button class="btn btn-light" data-toggle="collapse" data-target="#collapse1" aria-expanded="true" aria-controls="collapse1" style="text-decoration: none;">
                Light One
                </button>
                <button type="button" class="btn btn-default" onclick="turnOnLight(0)">
                <span class="material-icons" id="lightbulb0" style="color:rgb(243, 210, 155)">lightbulb</span>
                </button>
            </h5>
            </div>
            <div id="collapse1" class="collapse show" aria-labelledby="lightCard1" data-parent="#accordionlight">
                <div class="card-body">
                    <form>
                        <div class="form-row">
                            <div class="form-group col-1">
                                <label class="form-label">Light Color</label>
                                <input type="color" class="form-control form-control-color" id="light1-color" value="#ffffff" title="Choose light color">
                            </div>
                            <div class="form-group col-2">
                                <label>Shininess</label>
                                <select id="light1-shine" class="form-control">
                                    <option value="1">Very Weak</option>
                                    <option value="2">Weak</option>
                                    <option selected value="3">Moderate</option>
                                    <option value="4">Strong</option>
                                    <option value="5">Very Strong</option>
                                </select>
                            </div>
                            <div class="form-group col-3">
                                <label class="form-label">Light X Position</label>
                                <input type="number" class="form-control" id="light1-x" value="15" placeholder="15" data-bind="value:replyNumber">
                            </div>
                            <div class="form-group col-3">
                                <label class="form-label">Light Y Position</label>
                                <input type="number" class="form-control" id="light1-y" value="15" placeholder="15" data-bind="value:replyNumber">
                            </div>
                            <div class="form-group col-3">
                                <label class="form-label">Light Z Position</label>
                                <input type="number" class="form-control" id="light1-z" value="1" placeholder="1" data-bind="value:replyNumber">
                            </div>
                        </div>
                        <div class="form-row">  
                            <div class="form-group col-4">
                                <label class="form-label">Ambient Reflection (ka)</label>
                                <input type="range" id="light1-ka" class="custom-range" value="10" min="0" max="100" oninput="l1ka.value = this.value/100">
                                <small class="form-text text-secondary">
                                    Ambient reflection (ka) is <output name="l1ka">0.1</output>
                                </small>
                            </div>
                            <div class="form-group col-4">
                                <label class="form-label">Diffuse Reflection (kd)</label>
                                <input type="range" id="light1-kd" class="custom-range" value="60" min="0" max="100" oninput="l1kd.value = this.value/100">
                                <small class="form-text text-secondary">
                                    Diffuse reflection (kd) is <output name="l1kd">0.6</output>
                                </small>
                            </div>
                            <div class="form-group col-4">
                                <label class="form-label">Specular Reflection (ks)</label>
                                <input type="range" id="light1-ks" class="custom-range" value="30" min="0" max="100" oninput="l1ks.value = this.value/100">
                                <small class="form-text text-secondary">
                                    Specular reflection (ks) is <output name="l1ks">0.3</output>
                                </small>
                            </div>
                        </div>
                    </form>
                </div>
            </div>
        </div>
        <div class="card">
            <div class="card-header" id="lightCard2">
            <h5 class="mb-0 inline">
                <button class="btn btn-light collapsed" data-toggle="collapse" data-target="#collapse2" aria-expanded="false" aria-controls="collapse2" style="text-decoration: none;">
                    Light Two
                </button>
                <button type="button" class="btn btn-default" onclick="turnOnLight(1)">
                    <span class="material-icons" id="lightbulb1" style="color:rgb(0, 0, 0)">lightbulb</span>
                    </button>
            </h5>
            </div>
            <div id="collapse2" class="collapse" aria-labelledby="lightCard2" data-parent="#accordionlight">
                <div class="card-body">
                    <form>
                        <div class="form-row">
                            <div class="form-group col-1">
                                <label class="form-label">Light Color</label>
                                <input type="color" class="form-control form-control-color" id="light2-color" value="#f67e7d" title="Choose light color">
                            </div>
                            <div class="form-group col-2">
                                <label>Shininess</label>
                                <select id="light2-shine" class="form-control">
                                    <option value="1">Very Weak</option>
                                    <option value="2">Weak</option>
                                    <option selected value="3">Moderate</option>
                                    <option value="4">Strong</option>
                                    <option value="5">Very Strong</option>
                                </select>
                            </div>
                            <div class="form-group col-3">
                                <label class="form-label">Light X Position</label>
                                <input type="number" class="form-control" id="light2-x" value="-20" placeholder="-20" data-bind="value:replyNumber">
                            </div>
                            <div class="form-group col-3">
                                <label class="form-label">Light Y Position</label>
                                <input type="number" class="form-control" id="light2-y" value="-20" placeholder="-20" data-bind="value:replyNumber">
                            </div>
                            <div class="form-group col-3">
                                <label class="form-label">Light Z Position</label>
                                <input type="number" class="form-control" id="light2-z" value="0" placeholder="0" data-bind="value:replyNumber">
                            </div>
                        </div>
                        <div class="form-row">  
                            <div class="form-group col-4">
                                <label class="form-label">Ambient Reflection (ka)</label>
                                <input type="range" id="light2-ka" class="custom-range" value="10" min="0" max="100" oninput="l2ka.value = this.value/100">
                                <small class="form-text text-secondary">
                                    Ambient reflection (ka) is <output name="l2ka">0.1</output>
                                </small>
                            </div>
                            <div class="form-group col-4">
                                <label class="form-label">Diffuse Reflection (kd)</label>
                                <input type="range" id="light2-kd" class="custom-range" value="60" min="0" max="100" oninput="l2kd.value = this.value/100">
                                <small class="form-text text-secondary">
                                    Diffuse reflection (kd) is <output name="l2kd">0.6</output>
                                </small>
                            </div>
                            <div class="form-group col-4">
                                <label class="form-label">Specular Reflection (ks)</label>
                                <input type="range" id="light2-ks" class="custom-range" value="30" min="0" max="100" oninput="l2ks.value = this.value/100">
                                <small class="form-text text-secondary">
                                    Specular reflection (ks) is <output name="l2ks">0.3</output>
                                </small>
                            </div>
                        </div>
                    </form>
                </div>
            </div>
        </div>
        <div class="card">
            <div class="card-header" id="lightCard3">
            <h5 class="mb-0 inline">
                <button class="btn btn-light collapsed" data-toggle="collapse" data-target="#collapse3" aria-expanded="false" aria-controls="collapse3" style="text-decoration: none;">
                    Light Three
                </button>
                <button type="button" class="btn btn-default" onclick="turnOnLight(2)">
                    <span class="material-icons" id="lightbulb2" style="color:rgb(0, 0, 0)">lightbulb</span>
                    </button>
            </h5>
            </div>
            <div id="collapse3" class="collapse" aria-labelledby="lightCard3" data-parent="#accordionlight">
            <div class="card-body">
                <form>
                    <div class="form-row">
                        <div class="form-group col-1">
                            <label class="form-label">Light Color</label>
                            <input type="color" class="form-control form-control-color" id="light3-color" value="#843b62" title="Choose light color">
                        </div>
                        <div class="form-group col-2">
                            <label>Shininess</label>
                            <select id="light3-shine" class="form-control">
                                <option value="1">Very Weak</option>
                                <option value="2">Weak</option>
                                <option selected value="3">Moderate</option>
                                <option value="4">Strong</option>
                                <option value="5">Very Strong</option>
                            </select>
                        </div>
                        <div class="form-group col-3">
                            <label class="form-label">Light X Position</label>
                            <input type="number" class="form-control" id="light3-x" value="0" placeholder="0" data-bind="value:replyNumber">
                        </div>
                        <div class="form-group col-3">
                            <label class="form-label">Light Y Position</label>
                            <input type="number" class="form-control" id="light3-y" value="0" placeholder="0" data-bind="value:replyNumber">
                        </div>
                        <div class="form-group col-3">
                            <label class="form-label">Light Z Position</label>
                            <input type="number" class="form-control" id="light3-z" value="-100" placeholder="-100" data-bind="value:replyNumber">
                        </div>
                    </div>
                    <div class="form-row">  
                        <div class="form-group col-4">
                            <label class="form-label">Ambient Reflection (ka)</label>
                            <input type="range" id="light3-ka" class="custom-range" value="10" min="0" max="100" oninput="l3ka.value = this.value/100">
                            <small class="form-text text-secondary">
                                Ambient reflection (ka) is <output name="l3ka">0.1</output>
                            </small>
                        </div>
                        <div class="form-group col-4">
                            <label class="form-label">Diffuse Reflection (kd)</label>
                            <input type="range" id="light3-kd" class="custom-range" value="60" min="0" max="100" oninput="l3kd.value = this.value/100">
                            <small class="form-text text-secondary">
                                Diffuse reflection (kd) is <output name="l3kd">0.6</output>
                            </small>
                        </div>
                        <div class="form-group col-4">
                            <label class="form-label">Specular Reflection (ks)</label>
                            <input type="range" id="light3-ks" class="custom-range" value="30" min="0" max="100" oninput="l3ks.value = this.value/100">
                            <small class="form-text text-secondary">
                                Specular reflection (ks) is <output name="l3ks">0.3</output>
                            </small>
                        </div>
                    </div>
                </form>
            </div>
            </div>
        </div>
    </div>
</div>

    
