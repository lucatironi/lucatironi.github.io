---
layout: post
title: Manage dependencies with CMake FetchContent for C++ and OpenGL projects
tagline: How to setup Visual Studio Code and CMake to automatically handle the dependencies of your OpenGL and C++ projects.
description: "Developing a Graphics Engine with OpenGL and C++ for your games is fun but requires lots of dependencies: by using CMake, the FetchContent module and Visual Studio Code you can simplify the setup and compilation of your projects."
category: tutorial
tags: [OpenGL, C++, CMake, FetchContent, glfw, glad, glm, Visual Studio Code]
---

*\[Disclaimer: If you don't want to follow the tutorial and directly look at the code, you can find an advanced version on my [GitHub](https://github.com/lucatironi/cpp-gl-base)\]*

Over the years I tried to write my own little graphics engine using C++ and OpenGL. To do so I started using [GLFW](https://www.glfw.org/) to handle the multi-platform window creation, the input/events management together with other libraries and utilities like [glad](https://glad.dav1d.de/) and [glm](https://glm.g-truc.net/).

I am admittedly not very good with C++ and especially in dealing with its complex dependency management (or lack of thereof) and the building environment, so I always wanted to have the simplest way to keep my projects in check and manage the libraries and dependencies in the easiest way.

After having spent years doing it "wrong", first downloading the source files for the libraries and later at least using git submodules to automate their downloads and updates, I found out about CMake and its <code>FetchContent</code> module.

## CMake and FetchContent

[CMake](https://cmake.org/) is a build-system generator that using a Domain System Language (DSL) allows you to describe what and how to build and compile your code. The <code>FetchContent</code> module provides some ways to automatically manage the dependencies of your code, fetching them from other local or remote repositories and defining how to build and integrate them in your code.

CMake is platform and compiler agnostic, it just provides scripts that can be used on several operative systems and can be integrated in your development environment and with your tools. For this reason, it helped me to create projects that can be easily ported and developed on different machines, running MacOSX or Windows without any problem.

By using an IDE like Visual Studio Code, you can leverage several extensions that use your CMake scripts to automatically configure, compile and run your projects.

In this short tutorial I want to show you how I used these tools to quickly set up a C++ project that can be automatically compiled by VSCode with all the needed dependencies fetched from remote GIT repositories.

I start by creating a simple <code>CMakeLists.txt</code> file that contains our settings for CMake in order to generate the build system that VSCode can use to compile the project.

{% highlight cmake %}
# File: CMakeLists.txt
cmake_minimum_required(VERSION 3.16)

project(cpp-gl-base)

add_subdirectory(deps)

add_executable(${PROJECT_NAME} main.cpp)

target_link_libraries(${PROJECT_NAME} PRIVATE glfw glad glm)
{% endhighlight %}

As you can see, after having told which minimum version of CMake to use, I defined the name of my project (<code>cpp-gl-base</code>), told where my dependencies will be listed, which file to compile (<code>main.cpp</code>) and which libraries are needed to be linked with it. Nothing too complicated so far, especially for such a simple situation like this with just one single source file to take care.

The other <code>CMakeLists.txt</code> lives in the <code>deps/</code> directory mainly for the sake of separating the concerns between the one dedicated to our source files and the one for the dependencies. In theory this content could be pasted instead of the <code>add_subdirectory(deps)</code> and it will work as well.

{% highlight cmake %}
# File: deps/CMakeLists.txt
include(FetchContent)

# glfw
FetchContent_Declare(
    glfw
    GIT_REPOSITORY https://github.com/glfw/glfw.git
    GIT_TAG 3.4)

FetchContent_MakeAvailable(glfw)

# glad
FetchContent_Declare(
    glad
    GIT_REPOSITORY https://github.com/Dav1dde/glad.git
    GIT_TAG v0.1.36)

FetchContent_MakeAvailable(glad)

# glm
FetchContent_Declare(
    glm
    GIT_REPOSITORY https://github.com/g-truc/glm.git
    GIT_TAG 1.0.1)

FetchContent_MakeAvailable(glm)
{% endhighlight %}

Here we start by including the <code>FetchContent</code> module that provides two directives: <code>FetchContent_Declare</code> tells where to get the content to be fetched from and <code>FetchContent_MakeAvailable</code> to add it to the content in your build system.

As you can see in the case of our three libraries, we use a GitHub repository but it can be a local or remote archive or another source control repo. By defining a <code>GIT_TAG</code> we can clamp our dependencies to a specific version, streamlining the stabilization of the codebase and easing any update.

If you want to learn more about this module, you can check the [FetchContent Documentation](https://cmake.org/cmake/help/latest/module/FetchContent.html) and this [CMake Workshop](https://coderefinery.github.io/cmake-workshop/fetch-content/).

## The main.cpp

Now that we have created the two CMakeLists.txt files, we can add our source code to the directory and compile our project. You should have a directory structure like this:

{% highlight bash %}
  cpp-gl-base/
  ├── deps/
  │   └── CMakeLists.txt
  ├── CMakeLists.text
  └── main.cpp
{% endhighlight %}

To test our project with some real C++ and OpenGL code, I took the simple [GLFW example](https://www.glfw.org/documentation.html) on the official library's documentation and just added a couple of more lines to use <code>glad</code> to load the OpenGL function pointers. The code just inits the framework, instantiates a 640x480 window managed by GLFW, establishes a basic render loop that clear and swap the buffers and listens to input and window events.

{% highlight c++ %}
// File: src/main.cpp
#include <glad/glad.h>
#include <GLFW/glfw3.h>

int main(void)
{
    GLFWwindow* window;

    /* Initialize the library */
    if (!glfwInit())
        return -1;

    /* Create a windowed mode window and its OpenGL context */
    window = glfwCreateWindow(640, 480, "Hello World", NULL, NULL);
    if (!window)
    {
        glfwTerminate();
        return -1;
    }

    /* Make the window's context current */
    glfwMakeContextCurrent(window);

    /* glad: load all OpenGL function pointers */
    if (!gladLoadGLLoader((GLADloadproc)glfwGetProcAddress))
        return -1;

    /* Loop until the user closes the window */
    while (!glfwWindowShouldClose(window))
    {
        /* Render here */
        glClearColor(0.2f, 0.3f, 0.3f, 1.0f);
        glClear(GL_COLOR_BUFFER_BIT);

        /* Swap front and back buffers */
        glfwSwapBuffers(window);

        /* Poll for and process events */
        glfwPollEvents();
    }

    glfwTerminate();
    return 0;
}
{% endhighlight %}

If you want to know more about graphics and game programming, there are plenty of documentation and tutorials on the web to learn game engine development with C++ and OpenGL; my favorite one is [LearnOpenGL](https://learnopengl.com/) and it uses GLFW, glad and glm as well.

## Visual Studio Code

If you open this project's directory in Visual Studio Code, it should automatically ask you to download some extensions to manage the CMake workflow for you. Let it install them and VSCode will configure, download and compile the dependencies we listed in our scripts.

VSCode will ask you to choose a "build kit" and depending on your operative system and configuration, you can choose your compiler of choice. In my case - on a MacBook - I have installed and configured <code>Clang</code> but your mileage may vary.

![cpp-gl-base VSCode Setup](/assets/uploads/images/cpp-gl-base_vscode_1.png)

At the bottom of the IDE you will see a "play" button (circled in red on the picture above): by clicking it VSCode will compile your project and launch it, opening the executable and showing you the awesome little "Hello World" OpenGL window ready to display your game.

![cpp-gl-base VSCode Result](/assets/uploads/images/cpp-gl-base_vscode_2.png)

## Conclusions

This little tutorial should just touch the surface of many topics and it is not intended to be an exhaustive guide on how to develop graphical engine or games in C++ and OpenGL. Nevertheless I hope it provides you an entry point to start with a good base and be able to continue your journey with solid foundations when it comes to dependencies management.

I created a [repo on GitHub](https://github.com/lucatironi/cpp-gl-base) that contains a little more advanced <code>main.cpp</code> code and with more libraries added as dependencies in the CMakeLists.txt file: [Assimp](https://www.assimp.org/) for 3D assets importing, [stb_image](https://github.com/nothings/stb) for image loading/decoding from file/memory and [FreeType](https://freetype.org/) to render text. Check it out if you are interested in seeing the next ideal step after this tutorial!

Thanks for your attention and have fun!

Luca