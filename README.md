import cv2
import mediapipe as mp
import numpy as np
import tkinter as tk
from tkinter import colorchooser
from threading import Thread

# ---------------- Mediapipe Setup ---------------- #
mp_hands = mp.solutions.hands
mp_draw = mp.solutions.drawing_utils
hands = mp_hands.Hands(max_num_hands=1, min_detection_confidence=0.7)

# ---------------- Globals ---------------- #
brush_color = (0, 0, 255)   # Default Red
brush_size = 5
drawing = True
canvas = np.ones((480, 640, 3), np.uint8) * 255
prev_x, prev_y = None, None

# Predefined colors (BGR)
color_boxes = {
    "Blue": ((10, 10, 80, 60), (255, 0, 0)),
    "Green": ((90, 10, 160, 60), (0, 255, 0)),
    "Red": ((170, 10, 240, 60), (0, 0, 255)),
    "Yellow": ((250, 10, 320, 60), (0, 255, 255)),
    "Black": ((330, 10, 400, 60), (0, 0, 0)),
    "Eraser": ((410, 10, 500, 60), "eraser")  # special eraser flag
}

# ---------------- OpenCV Drawing Loop ---------------- #
def draw_on_canvas():
    global canvas, brush_color, brush_size, drawing, prev_x, prev_y

    cap = cv2.VideoCapture(0)

    while True:
        ret, frame = cap.read()
        if not ret:
            break

        frame = cv2.flip(frame, 1)
        rgb = cv2.cvtColor(frame, cv2.COLOR_BGR2RGB)
        results = hands.process(rgb)

        # Draw color buttons on frame
        for name, (coords, color) in color_boxes.items():
            x1, y1, x2, y2 = coords
            draw_color = (255, 255, 255) if color == "eraser" else color
            cv2.rectangle(frame, (x1, y1), (x2, y2), draw_color, -1)
            cv2.putText(frame, name, (x1 + 5, y2 + 25), cv2.FONT_HERSHEY_SIMPLEX, 0.5,
                        (0,0,0) if color==(255,255,255) or color=="eraser" else (255,255,255), 2)

        if results.multi_hand_landmarks:
            for handLms in results.multi_hand_landmarks:
                mp_draw.draw_landmarks(frame, handLms, mp_hands.HAND_CONNECTIONS)

                h, w, c = frame.shape
                x = int(handLms.landmark[8].x * w)
                y = int(handLms.landmark[8].y * h)

                cv2.circle(frame, (x, y), 5, (0, 255, 0), -1)

                # Check if touching any color box
                for name, (coords, color) in color_boxes.items():
                    x1, y1, x2, y2 = coords
                    if x1 <= x <= x2 and y1 <= y <= y2:
                        if color == "eraser":
                            brush_color = "eraser"
                        else:
                            brush_color = color
                        prev_x, prev_y = None, None

                # Drawing
                if drawing:
                    if prev_x is not None and prev_y is not None:
                        if brush_color == "eraser":
                            cv2.line(canvas, (prev_x, prev_y), (x, y), (255, 255, 255), brush_size * 3)
                        else:
                            cv2.line(canvas, (prev_x, prev_y), (x, y), brush_color, brush_size * 2)
                    prev_x, prev_y = x, y
                else:
                    prev_x, prev_y = None, None
        else:
            prev_x, prev_y = None, None

        # Show separate windows
        cv2.imshow("Webcam Output", frame)
        cv2.imshow("Paint Canvas", canvas)

        key = cv2.waitKey(1) & 0xFF
        if key == ord('q'):
            break

    cap.release()
    cv2.destroyAllWindows()

# ---------------- Tkinter UI Functions ---------------- #
def choose_color():
    global brush_color
    color_code = colorchooser.askcolor(title="Choose Brush Color")
    if color_code[0]:
        brush_color = tuple(map(int, color_code[0][::-1]))

def change_brush(val):
    global brush_size
    brush_size = int(val)

def toggle_drawing():
    global drawing
    drawing = not drawing
    status.config(text="Drawing: ON" if drawing else "Drawing: OFF")

def clear_canvas():
    global canvas
    canvas = np.ones((480, 640, 3), np.uint8) * 255

def save_canvas():
    global canvas
    cv2.imwrite("AirCanvasUltra.png", canvas)
    print("✅ Saved as AirCanvasUltra.png")

def set_color(color):
    global brush_color
    brush_color = color

# ---------------- Tkinter UI ---------------- #
root = tk.Tk()
root.title("Air Canvas Ultra")
root.geometry("300x450")

tk.Button(root, text="Choose Any Color", command=choose_color).pack(pady=10)

# Predefined color buttons
tk.Button(root, text="Blue", bg="blue", fg="white", command=lambda:set_color((255,0,0))).pack(fill="x")
tk.Button(root, text="Green", bg="green", fg="white", command=lambda:set_color((0,255,0))).pack(fill="x")
tk.Button(root, text="Red", bg="red", fg="white", command=lambda:set_color((0,0,255))).pack(fill="x")
tk.Button(root, text="Yellow", bg="yellow", fg="black", command=lambda:set_color((0,255,255))).pack(fill="x")
tk.Button(root, text="Black", bg="black", fg="white", command=lambda:set_color((0,0,0))).pack(fill="x")
tk.Button(root, text="Eraser", bg="white", fg="black", command=lambda:set_color("eraser")).pack(fill="x")

tk.Label(root, text="Brush Size").pack()
tk.Scale(root, from_=1, to=20, orient="horizontal", command=change_brush).pack()

status = tk.Label(root, text="Drawing: ON")
status.pack(pady=5)

tk.Button(root, text="Toggle Draw", command=toggle_drawing).pack(pady=10)
tk.Button(root, text="Clear Canvas", command=clear_canvas).pack(pady=10)
tk.Button(root, text="Save Canvas", command=save_canvas).pack(pady=10)

# Start drawing thread automatically
Thread(target=draw_on_canvas, daemon=True).start()

root.mainloop()
