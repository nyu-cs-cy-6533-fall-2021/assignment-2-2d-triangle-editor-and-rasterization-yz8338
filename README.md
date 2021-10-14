# CS-GY 6533 A – Interactive Computer Graphics - Fall 2021

### Assignment 2

*Yangfan Zhou*

<yz8338@nyu.edu>

# Implementation & Result

## Triangle Soup Editor (Task 1.1)

* Insertion mode (i): 

Firstly, detect whether the mode is active by capturing GLFW_KEY_I in key_callback function. If the 'i' key is pressed, we set the variable 'iKey' to true. Then enlarge the vector size of V and C by incrementing 3. 
```bash
case GLFW_KEY_I:
    if (action == GLFW_PRESS) {
        iKey = true;
        V.resize(V.size() + 3);
        C.resize(C.size() + 3);
        cout << "triangle insertion mode start \n";
    }
    break;
```

We begin drawing triangle with preview in the main render loop then. Detect if the insertion mode is on by checking 'iKey'; we also use variable 'insertIndex' which starts at 0 and record the inserted number of points to draw triangles. If the 'insertIndex' can be divided by 3, we draw point; if it divides 3 and the remainder is 1, we draw line; otherwise, draw the border of triangle. The color of the inserting point is set to black at the beginning and the position of point is obtained by 'getCurrentWorldPos' function which I defined before.
```bash
// in render loop
if (iKey) {
    V[insertIndex] = getCurrentWorldPos(window);
    VBO.update(V); 

    C[insertIndex] = glm::vec3(0, 0, 0);
    VBO_C.update(C);

    if (insertIndex % 3 == 0) {
        glPointSize(3.f);
        glDrawArrays(GL_POINTS, insertIndex, 1);
    } else if (insertIndex % 3 == 1) {
        glLineWidth(3.f);
        glDrawArrays(GL_LINES, insertIndex - 1, 2);
    } else if (insertIndex % 3 == 2) {
        glDrawArrays(GL_LINE_LOOP, insertIndex - 2, 3);
    }
}
```
```bash
glm::vec2 getCurrentWorldPos(GLFWwindow* window) {
    // Get the position of the mouse in the window
    double xpos, ypos;
    glfwGetCursorPos(window, &xpos, &ypos);

    // Get the size of the window
    int width, height;
    glfwGetWindowSize(window, &width, &height);

    // Convert screen position to world coordinates
    glm::vec4 p_screen(xpos,height-1-ypos,0,1);
    glm::vec4 p_canonical((p_screen.x/width)*2-1,(p_screen.y/height)*2-1,0,1);
    glm::vec4 p_world = glm::inverse(view) * p_canonical;

    return glm::vec2(p_world.x, p_world.y);
}
```

The 'insertIndex' and all the points of triangle is determined by catching mouse click event in mouse_button_callback function. If iKey is true and the mouse is clicked, we assign the current position of cursor into the current point. Then increment the recorder 'insertIndex' by 1. Next, we determine whether it is the third point of the current triangle, if it is, then set iKey to false and set the color of all points in this triangle to red by 'resetColor' function that I defined.
```bash
void resetColor(int i_triangle, float r, float g, float b) {
    int t = i_triangle * 3;

    for (int i = 0; i < 3; i ++) {
        C[t + i] = glm::vec3(r, g, b);       
    }

    VBO_C.update(C);
}
```
```bash
    // Triangle insertion mode (in mouse_button_callback)
    if (iKey && action == GLFW_PRESS) {
        V[insertIndex] = getCurrentWorldPos(window);
        insertIndex += 1;
        if (insertIndex >= V.size() - 3) {
            iKey = false;
            resetColor(insertIndex/3 - 1, 1.0, 0.0, 0.0);
        }
    }
```

The triangles is displayed all the time by the following code in main loop. We first draw all triangles in its color, and then for all triangle, we reset vertex color to black and draw border by GL_LINE_LOOP for them. The program record the original color at first and then re-assign the original color back after drawing the border.
```bash
// in render loop
glDrawArrays(GL_TRIANGLES, 0, insertIndex);
     
for (int i = 0; i < insertIndex / 3; i ++) {
    temp_C = C;
    resetColor(i, 0.0, 0.0, 0.0);
    glDrawArrays(GL_LINE_LOOP, i * 3, 3);
    C = temp_C;
    VBO_C.update(C);
}
```
Note that VBO_C is another VertexBufferObject for color buffer and C is another vectors to store the per-vertex color.

* Translation mode (o): 

Similar to insertion mode, this beginning of this mode is also detected in key_callback as follows:
```bash
case GLFW_KEY_O:
    if (action == GLFW_PRESS) {
        oKey = true;
        cout << "triangle translation mode start \n";
    }
    break;
```

If the mode is active, we detect the press, move and release activities in mouse_button_callback function. If 'oKey' is true and the mouse is pressing, we record the current cursor position by getCurrentWorldPos function. Then use the cursor's position to detect whether there is a triangle under the cursor by 'getCurrentTriangle(cursor)'. If there is no triangle, the return value is -1. If not, we set the variable 'drag' to true and use a temporary vector 'ColorBefore' to store the current colors. Because we need to highlight the triangle as blue when dragging the selected one. 
If the mode is active and the mouse is release, we set 'drag' and 'oKey' to false. Then retrieve the original colors of the triangle.
```bash
// in mouse_button_callback
// Triangle translation mode on
    if (oKey && action == GLFW_PRESS) {
        cursor = getCurrentWorldPos(window);
        triangle = getCurrentTriangle(cursor); // select triangle
        if (triangle != -1) {
            drag = true;
            ColorBefore = C;
        }
    } else if (oKey && action == GLFW_RELEASE) {
        drag = false;
        oKey = false;
        // retrieve original color
        int index = triangle * 3;
        C[index] = ColorBefore[index];
        C[index + 1] = ColorBefore[index + 1];
        C[index + 2] = ColorBefore[index + 2];
    }
```

For the 'getCurrentTriangle' function, it traverse all triangle in vector V and use the current cursor position with 'pointInTriangle' function to detect whether the cursor is upon a triangle. If so, return the index of triangle. If not, return -1.
```bash
int getCurrentTriangle(glm::vec2 cursor) {
    for (int i = 0; i < insertIndex / 3; i ++) {
        //glDrawArrays(GL_LINE_LOOP, i * 3, 3);
        int index = i * 3;
        bool temp = pointInTriangle(V[index][0], V[index][1], V[index+1][0], V[index+1][1], V[index+2][0], V[index+2][1], cursor[0], cursor[1]);
        if (temp) 
            return i;
    }
    return -1;
}
```

For the 'pointInTriangle' function, we use barycentric theory to determine whether a point is in given triangle. Then return the result as boolean value.
```bash
bool pointInTriangle(float x1, float y1, float x2, float y2, float x3, float y3, float x, float y) {
    float denominator = ((y2 - y3)*(x1 - x3) + (x3 - x2)*(y1 - y3));
    float a = ((y2 - y3)*(x - x3) + (x3 - x2)*(y - y3)) / denominator;
    float b = ((y3 - y1)*(x - x3) + (x1 - x3)*(y - y3)) / denominator;
    float c = 1 - a - b;

    return 0 <= a && a <= 1 && 0 <= b && b <= 1 && 0 <= c && c <= 1;
}
```

Finally, we detect whether 'drag' is true in the main loop. If the cursor is dragging, we use 'translateTriangle' function to update the real-time position of selected triangle. The first parameter 'triangle' is just the index returned from the getCurrentTriangle function. 
```bash
// in render loop
// Translation Mode
    if (oKey) {
       if (drag) {
           translateTriangle(triangle, window); // highlight & translation
       }
    }
```

In the 'translateTriangle' function, we get current cursor's position first. In order to calculate the movement of cursor, we use the current position to subtract the old position (which is recorded in global variable 'cursor'). Then, for each vertex in the triangle, we increment its coordinate by the movement of cursor. Simultaneously, change the vertex color to blue in order to highlight the triangle. In the end of this function, we record the current cursor position into 'cursor'. Finally, update the current vertex vector V and color vector C.
```bash
void translateTriangle(int index, GLFWwindow* window) {
    
    glm::vec2 temp = getCurrentWorldPos(window);
    float trans_x = temp[0] - cursor[0];
    float trans_y = temp[1] - cursor[1];

    int t = index * 3;

    for (int i = 0; i < 3; i ++) {
        V[t + i][0] = V[t + i][0] + trans_x;
        V[t + i][1] = V[t + i][1] + trans_y;
        C[t + i] = glm::vec3(0, 0, 1); // highlight as blue      
    }

    cursor = getCurrentWorldPos(window);

    VBO.update(V);
    VBO_C.update(C);
}
```

* Deletion mode (p):

Also similar as before, the deletion mode is first detected in key_callback function with GLFW_KEY_P. If the 'k' key is pressed, we set the variable 'pKey' to true. The section is as follows:
```bash
case GLFW_KEY_P:
    if (action == GLFW_PRESS) {
        pKey = true;
        cout << "triangle delete mode start \n";
    }
    break;
```

Next, we detect whether the user click certain triangle in this mode in mouse_button_callback function. If 'pKey' is true and the mouse is clicked, obtain the current cursor position and pass it to getCurrentTriangle function. The getCurrentTriangle function is the same as the one in translation mode, and is explained before. After retrieving current triangle's index under the cursor, we pass the index into deleteTriangle function. Finally, set 'pKey' to false after deletion.
```bash
// in mouse_button_callback
// Triangle deletion mode on
    if (pKey && action == GLFW_PRESS) {
        cursor = getCurrentWorldPos(window);
        triangle = getCurrentTriangle(cursor); // select triangle
        deleteTriangle(triangle); // delete triangle
        pKey = false;
    }
```

For the deleteTriangle function, it first check whether the triangle index is valid, i.e. there is a triangle under the cursor. If not, the function do nothing. If true, it first pop the last three element of the vector (because I set the vector V's original size as 3). Then, it use vector's erase method to delete the triangle at the position of index (the index is triangle index, we need to first multiply it by 3 to get the first vertex index). Finally, resize the V vector and set the current triangle index as -1.
```bash
void deleteTriangle(int index) {
    if (index > -1) {
        for (int i = 0; i < 3; i ++) {
            V.pop_back();
        }
        int temp = index * 3;
        V.erase(V.begin() + temp, V.begin() + temp + 3);

        insertIndex = (int)V.size();
        V.resize(V.size() + 3);

        triangle = -1;
    }
}
```

## Rotation/Scale (Task 1.2)

* Rotation (h & j):

Rotation mode is detected in key_callback function at first. For h & j key, if the key is pressed and any triangle is selected, we pass the current triangle index into 'rotate' function to rotate the current triangle. As we mentioned before, after translation mode, the global variable 'triangle' isn't changed. Hence, if any triangle has been selected by translation mode, the 'triangle' variable will be greater than -1. However, for other mode like deletion mode, we reset this variable to -1 after finishing deletion.
```bash
// in key_callback
// 'h': rotate 10 degree clockwise
    case GLFW_KEY_H:
        if (action == GLFW_PRESS && triangle > -1) {
            rotate(triangle, -10.0);
            cout << "triangle rotate 10 degree clockwise \n";
        }
        break;

// 'j': rotate 10 degree counter-clockwise
    case GLFW_KEY_J:
        if (action == GLFW_PRESS && triangle > -1) {
            rotate(triangle, 10.0);
            cout << "triangle rotate 10 degree counter-clockwise \n";
        }
        break;
```

In rotate function, we pass two variable that is triangle index and the rotation degree. To begin with, calculate the first vertex index of the triangle and assign it to temporary variable 't'. Then calculate the barycenter for the current triangle using the vertex indexes. Before implement the rotation transformation, transfer the degree into radian. Finally, for every vertex of this triangle, translate it to the origin point, then multiply it with rotation matrix, after that translate it back by adding the position of barycenter. By doing this, the triangle can rotate around the barycenter. Update the current vector V by VBO.update(V).
```bash
void rotate(int triangle, double degree) {
    
    int t = triangle * 3;

    // barycenter
    float xg = (V[t][0] + V[t+1][0] + V[t+2][0]) / 3.0f;
    float yg = (V[t][1] + V[t+1][1] + V[t+2][1]) / 3.0f;

    degree = degree * glm::pi<float>() / 180.0;

    for (int i = 0; i < 3; i ++) {
        V[t + i] = V[t + i] - glm::vec2(xg, yg);
        V[t + i] = V[t + i] * glm::mat2(cos(degree), -sin(degree), sin(degree), cos(degree));
        V[t + i] = V[t + i] + glm::vec2(xg, yg);
    }

    VBO.update(V);
}
```

* Scale (k & l):

Similar to rotation mode, we first detect the key k & l within key_callback function. If the key is pressed and there is triangle selected before in translation mode, call 'scale' function.
```bash
// in key_callback
// 'k': scale up 25%
    case GLFW_KEY_K:
        if (action == GLFW_PRESS && triangle > -1) {
            scale(triangle, 1.25);
            cout << "triangle scale up 25% \n";
        }
        break;

// 'l': scale down 25%
    case GLFW_KEY_L:
        if (action == GLFW_PRESS && triangle > -1) {
            scale(triangle, 0.75);
            cout << "triangle scale down 25% \n";
        }
        break;
```

For the scale function, similar as rotate function, it first derive the vertexes' index and calculate the barycenter coordinate. Then, for every vertex in the triangle, move it towards origin point by subtracting barycenter's position, next scale it by multiplying the scaler, finally translate it back by adding barycenter's position. Update the current vertex vector V by VBO.update(V).
```bash
void scale(int triangle, float perc) {

    int t = triangle * 3;

    // barycenter
    float xg = (V[t][0] + V[t+1][0] + V[t+2][0]) / 3.0f;
    float yg = (V[t][1] + V[t+1][1] + V[t+2][1]) / 3.0f;

    for (int i = 0; i < 3; i ++) {
        V[t + i] = V[t + i] - glm::vec2(xg, yg);
        V[t + i] = V[t + i] * glm::vec2(perc, perc);
        V[t + i] = V[t + i] + glm::vec2(xg, yg);
    }

    VBO.update(V);
}
```

## Colors (Task 1.3)

For this task, I changed the vertex_shader and fragment_shader with additional color vector, which is connected by the VBO_C buffer.
```bash
Program program;
    const GLchar* vertex_shader =
            "#version 150 core\n"
                    "in vec2 position;"
                    "in vec3 color;"
                    "out vec3 f_color;"
                    "uniform mat4 view;"
                    "void main()"
                    "{"
                    "    gl_Position = view * vec4(position, 0.0, 1.0);"
                    "    f_color = color;"
                    "}";
    const GLchar* fragment_shader =
            "#version 150 core\n"
                    "in vec3 f_color;"
                    "out vec4 outColor;"
                    "uniform vec3 triangleColor;"
                    "void main()"
                    "{"
                    "    outColor = vec4(f_color, 1.0);"
                    "}";
    ......
    ......
    program.bindVertexAttribArray("color",VBO_C);
```

* Color mode (c):

The color mode is detected by GLFW_KEY_C in key_callback function. If the 'c' key is pressed, we set the variable 'cKey' to true. After that, every press betweeen 1 to 9 can enable the color change activity. The point which requires change is recorded in the global variable 'closest', which is detected and assigned in 'getClosestVertex' function when mouse click event happened under color mode. We just change the the 'closest' point's color within color vector C directly to different colors.
```bash
    // in key_callback
    // 'c': color mode
        case GLFW_KEY_C:
            if (action == GLFW_PRESS) {
                cKey = true;
                cout << "color mode \n";
            }
            break;

        // '1' ~ '9': different color
        case GLFW_KEY_1:
            if (cKey) {
                C[closest] = glm::vec3(1.0, 0.0, 0.0); // red
                cKey = false;
            }
            break;

        case GLFW_KEY_2:
            if (cKey) {
                C[closest] = glm::vec3(0.0, 1.0, 0.0); // green
                cKey = false;
            }
            break;

        case GLFW_KEY_3:
            if (cKey) {
                C[closest] = glm::vec3(0.0, 0.0, 1.0); // blue
                cKey = false;
            }
            break;
        
        case GLFW_KEY_4:
            if (cKey) {
                C[closest] = glm::vec3(0.0, 1.0, 1.0); // cyan
                cKey = false;
            }
            break;

        case GLFW_KEY_5:
            if (cKey) {
                C[closest] = glm::vec3(1.0, 0.0, 1.0); // magenta
                cKey = false;
            }
            break;
        
        case GLFW_KEY_6:
            if (cKey) {
                C[closest] = glm::vec3(1.0, 1.0, 0.0); // yellow
                cKey = false;
            }
            break;

        case GLFW_KEY_7:
            if (cKey) {
                C[closest] = glm::vec3(0.5, 0.5, 1.0); // lighter blue
                cKey = false;
            }
            break;

        case GLFW_KEY_8:
            if (cKey) {
                C[closest] = glm::vec3(0.5, 1.0, 0.5); // lighter green
                cKey = false;
            }
            break;

        case GLFW_KEY_9:
            if (cKey) {
                C[closest] = glm::vec3(1.0, 0.5, 0.5); // lighter red
                cKey = false;
            }
            break;
```

The mouseclick is detected in mouse_button_callback function as well. If the color mode is enabled and the mouse is clicked, we get current cursor position by getCurrentWorldPos function. Then pass the coordinate into getClosestVertex. The result will be passed back to the global variable 'closest', which is used to change vertex color.
```bash
// in mouse_button_callback
// Color mode
    if (cKey && action == GLFW_PRESS) {
        cursor = getCurrentWorldPos(window);
        closest = getClosestVertex(cursor);
    }
```

For the getClosestVertex function, we traverse all the vertex in vector V and calculate their distance to cursor position, then find the minimal one, assign the vertex's index to 'result' and return it.
```bash
int getClosestVertex(glm::vec2 cursor) {
    float x = cursor[0];
    float y = cursor[1];

    float temp = 0;
    float dist;
    int result;

    for (int i = 0; i < V.size(); i ++) {
        dist = pow((V[i][0]-x), 2) + pow((V[i][1]-y), 2);
        if (temp) {
            if (dist < temp) {
                temp = dist;
                result = i;
            }
        } else {
            temp = dist;
            result = i;
        }
    }

    return result;
}
```

## View Control (Task 1.4)

For this task, I create a matrix for view transformation and two other vectors viewScale and viewTrans to change the view matrix.
```bash
// Contains the view transformation
glm::mat4 view;
glm::vec3 viewScale = glm::vec3(1.f, 1.f, 1.f);
glm::vec3 viewTrans = glm::vec3(0.f, 0.f, 0.f);
```

Then, I multiply the view matrix with original vertex within vertex_shader. 
```bash
const GLchar* vertex_shader =
        "#version 150 core\n"
                "in vec2 position;"
                "in vec3 color;"
                "out vec3 f_color;"
                "uniform mat4 view;"
                "void main()"
                "{"
                "    gl_Position = view * vec4(position, 0.0, 1.0);"
                "    f_color = color;"
                "}";
```

* Zoom in & out (+ & -)

Similar as before, the key + & - is detected within key_callback function. If + is pressed, we increment the viewScale vector by 0.2. If - is pressed, we decrement viewScale vector by 0.2f.
```bash
        // in key_callback
        // '+': zooming in 20%
        case GLFW_KEY_EQUAL:
            if (action == GLFW_PRESS) {
                viewScale += 0.2f;
                cout << "zoom in 20% \n";
            }
            break;

        // '-': zooming out 20%
        case GLFW_KEY_MINUS:
            if (action == GLFW_PRESS) {
                viewScale -= 0.2f;
                cout << "zoom out 20% \n";
            }
            break;
```

This viewScale vector is applied to the view matrix within the render loop by glm::scale.
```bash
view = glm::scale(glm::mat4(1.f), viewScale);
```

* Pan the view (w, a, s, d):

We change the viewTrans vector in order to translate the view as a whole. If w, a, s or d is pressed, change corresponding position element within the viewTrans vector.
```bash
        // 'w': pan the view 20% (down)
        case GLFW_KEY_W:
            if (action == GLFW_PRESS) {
                viewTrans[1] -= 0.2f;
                cout << "pan down 20% \n";
            }
            break;

        // 'a': pan the view 20% (right)
        case GLFW_KEY_A:
            if (action == GLFW_PRESS) {
                viewTrans[0] += 0.2f;
                cout << "pan right 20% \n";
            }
            break;

        // 's': pan the view 20% (up)
        case GLFW_KEY_S:
            if (action == GLFW_PRESS) {
                viewTrans[1] += 0.2f;
                cout << "pan up 20% \n";
            }
            break;

        // 'd': pan the view 20% (left)
        case GLFW_KEY_D:
            if (action == GLFW_PRESS) {
                viewTrans[0] -= 0.2f;
                cout << "pan left 20% \n";
            }
            break;
```

The viewTrans vector also applies to the view matrix in the render loop.
```bash
view = glm::translate(view, viewTrans);
```

* Change user interaction:

We ensure the user interaction adapted to the current view by applying the inverse of view matrix within getCurrentWorldPos function.
```bash
glm::vec2 getCurrentWorldPos(GLFWwindow* window) {
    // Get the position of the mouse in the window
    double xpos, ypos;
    glfwGetCursorPos(window, &xpos, &ypos);

    // Get the size of the window
    int width, height;
    glfwGetWindowSize(window, &width, &height);

    // Convert screen position to world coordinates
    glm::vec4 p_screen(xpos,height-1-ypos,0,1);
    glm::vec4 p_canonical((p_screen.x/width)*2-1,(p_screen.y/height)*2-1,0,1);
    glm::vec4 p_world = glm::inverse(view) * p_canonical;

    return glm::vec2(p_world.x, p_world.y);
}
```

## Add keyframing (Task 1.5)





# Compilation Instructions

```bash
cd Assignment_2
mkdir build
cd build
cmake ../ # re-run cmake when you add/delete source files
make # use "cmake --build ." for Windows
```
