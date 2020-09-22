---
layout: post
title: Porting the Chip8 emulator to the web
tags: chip8 emulation
---

[Emscripten](https://kripken.github.io/emscripten-site) is an LLVM-based compiler for converting C and C++ code into asm.js, a highly optimisable subset of JavaScript, which allows programs to run in a web browser at near native speed. This post will go through the steps required to port my Chip8 emulator to the web browser.

## Main Loop

Being my first emulation project and mainly done a learning exercise, my Chip8 emulator features a slightly... unusual architecture. Among other issues, its design made it difficult to compile to JavaScript using Emscripten. The emulator was originally written to run with two threads - one handling the emulation itself (CPU, graphics, audio) and the other handling the OpenGL interface. While support for shared memory multithreading has [recently been added](https://github.com/tc39/ecmascript_sharedmem) to the EMCAScript specification, there is no need for anything that complicated in a Chip8 emulator - it should happily run in a single thread on any machine.

Running the emulation in the application's main thread was as simple as moving the `step()` emulation function and the OpenGL rendering code into a single `run_frame()` function. The only difference is that now the game speed is dependent on the FPS the emulator is running at. However, since the Chip8 was never a real physical console, it doesn't have a defined speed to be run at and most games will require different speed settings anyway.

Assuming a frame rate of 60fps, running at 10 instructions per frame (or a little higher) works well for most Chip-8 games tested, but Connect4 needs to run at only 1 instruction per frame to be playable. Super-Chip games generally need to be run a bit faster - somewhere in the 20-60 range seems to work well.

Now that the emulator runs in a single thread, we can tell Emscripten what it should run as its main loop:

{% highlight cpp %}
#ifdef __EMSCRIPTEN__
  emscripten_set_main_loop_arg(run_frame, this, 0, 1);
#else
  while (!glfwWindowShouldClose(window))
  {
    run_frame(this);
    glfwSwapBuffers(window);
    glfwPollEvents();
  }
#endif
{% endhighlight %}

## WebGL

WebGL is based on OpenGL ES, which supports approximately a subset of the features of full OpenGL, so some slight modifications were needed to the OpenGL API calls and shaders.

### Shaders

In newer versions of OpenGL, the `attribute` and `varying` keywords have been replaced by `in` and `out`, with the choice of replacement depending on whether we're working on a vertex or fragment shader. To make our shaders work with WebGL, we'll have to downgrade them to use the older syntax. Additionally, WebGL requires that we specify a precision that we want to work with in the fragment shader.

Vertex shader:
- `in` -> `attribute`
- `out` -> `varying`

Fragment shader:
- `in` -> `varying`
- `out` -> gl_FragColor builtin


#### Old vertex shader

{% highlight glsl %}
#version 330 core
in vec2 position;
in vec2 texCoord;
out vec2 TexCoord;
void main()
{
  gl_Position = vec4(position, 0.0, 1.0);
  TexCoord = texCoord;
};
{% endhighlight %}

#### WebGL-compatible vertex shader

{% highlight glsl %}
attribute vec2 position;
attribute vec2 texCoord;
varying vec2 TexCoord;
void main()
{
  gl_Position = vec4(position, 0.0, 1.0);
  TexCoord = texCoord;
};
{% endhighlight %}

#### Old fragment shader

{% highlight glsl %}
#version 330 core
in vec2 TexCoord;
out vec4 colour;
uniform sampler2D display;
void main()
{
  colour = texture(display, TexCoord);
};
{% endhighlight %}

#### WebGL-compatible fragment shader

{% highlight glsl %}
#ifdef __EMSCRIPTEN__
precision mediump float;
#endif
varying vec2 TexCoord;
uniform sampler2D display;
void main()
{
  gl_FragColor = texture2D(display, TexCoord);
};
{% endhighlight %}

WebGL runs on a different set of version numbers to OpenGL, so I've just removed them entirely to make life easier.

### OpenGL API

Aside from some unnecessary OpenGL calls that could just be removed, only `glTexImage2D` had compatibility issues. This function tells the graphics card how to interpret the block of memory representing the Chip8's display output.

The Chip8 can only display two colours - black and white - so doesn't need to use the more standard `GL_RGB` / `GL_RGBA` formats. In OpenGL the pair `GL_RGBA8` / `GL_LUMINANCE` works to use a single byte as a greyscale colour output, but this doesn't work in WebGL. After some experimentation with different format combinations, the pair `GL_LUMINANCE` / `GL_LUMINANCE` was found to work in WebGL.

Unfortunately there doesn't seem to be a format which is compatible between OpenGL and WebGL, so we'll leave it up to the C++ preprocessor to pick which to use:

{% highlight cpp %}
#ifdef __EMSCRIPTEN__
  glTexImage2D(GL_TEXTURE_2D, 0, GL_LUMINANCE, w, h, 0, GL_LUMINANCE, GL_UNSIGNED_BYTE, disp);
#else
  glTexImage2D(GL_TEXTURE_2D, 0, GL_RGBA8, w, h, 0, GL_LUMINANCE, GL_UNSIGNED_BYTE, disp);
#endif
{% endhighlight %}

## JavaScript to C++ communication

The final step in porting the Chip8 emulator to the web browser was to enable communication from the JavaScript UI to the C++ core. This allows us to control settings in the emulator and, more importantly, to load games.

### Loading Games

Starting with the default HTML file Emscripten generates, we need to make a few changes so that our emulator's main function gets called with the arguments it needs.

First off, we'll say that we don't want the emulator to start as soon as the page is loaded by adding the following line in the definition of Emscripten's `Module`:

{% highlight js %}
noInitialRun: true,
{% endhighlight %}

Secondly, we need to add a file picker and some JavaScript to load the chosen ROM into the emulator:

{% highlight html %}
<input type="file" id="rom" />
{% endhighlight %}

{% highlight js %}
function loadROM(event) {
  var f = event.target.files[0];

  if (f) {
    var r = new FileReader();
    r.onload = function(e) {
      var contents = e.target.result;
      var file_name = document.getElementById("rom").files[0].name;
      FS.writeFile(file_name, new Uint8Array(contents), {encoding: "binary"});

      // Make sure any saved games are loaded before running
      FS.syncfs(true, function(err) {
        if (err) {
          console.error(err);
        } else {
          Module.callMain([file_name]);
        }
      });
    }
    r.readAsArrayBuffer(f);
    document.getElementById("rom").style.display = "none";
    document.getElementById("controls").style.display = "block";
  }
  else {
    alert("failed to load file\n");
  }
}

document.getElementById("rom").addEventListener("change", loadROM, false);
{% endhighlight %}

This code uses Emscripten's [file system API](https://kripken.github.io/emscripten-site/docs/api_reference/Filesystem-API.html) to save the chosen ROM into an in-memory filesystem, before calling the emulator's main function with the ROM's file path. This is necessary since the emulator expects to load ROMs from disk.

### Controlling game speed

In order to make all games playable, we need to provide a way of letting the user pick what speed they should run at.

We create a C function which can be called from JavaScript, and call it when the value in the "instructions per step" text box is updated:

{% highlight cpp %}
extern "C"
{
  void set_instructions_per_step(int n)
  {
    chip8.instructions_per_step = n;
  }
}
{% endhighlight %}

{% highlight html %}
Instructions per step: <input type="text" id="speed" />
{% endhighlight %}

{% highlight js %}
function updateSpeed() {
  var n = document.getElementById("speed").value;
  Module.ccall('set_instructions_per_step', 'void', '[number]', [n]);
}

document.getElementById("speed").addEventListener("input", updateSpeed, false);
{% endhighlight %}

## Try it out!

The JavaScript version of the Chip8 emulator is [running on this site](/chip8).
