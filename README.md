#Blink-Pointer
#An eye-controlled mouse system using Python, enabling hands-free interaction through blink detection.

# Initialize pygame mixer
pygame.mixer.init()
click_sound = pygame.mixer.Sound("click.mp3")  # Make sure 'click.mp3' is in the same folder or update the path

# MediaPipe Setup
mp_face_mesh = mp.solutions.face_mesh
face_mesh = mp_face_mesh.FaceMesh(
    max_num_faces=1,
    refine_landmarks=True,
    min_detection_confidence=0.6,
    min_tracking_confidence=0.6
)

# Globals
screen_w, screen_h = pyautogui.size()
prev_mouse_x, prev_mouse_y = pyautogui.position()
blink_threshold = 0.01
click_cooldown = 1
last_click_time = 0
last_blink_time = 0
blink_start_time = None
eye_history_length = 3
double_blink_window = 1.0
tracking_enabled = False
calibrated = False
overlay_enabled = True
blink_count = 0
eye_history_left = []
eye_history_right = []

APP_CONFIG_PATH = "app_config.json"

# Default apps grouped with full paths where possible

APPS = {

    "Browsers": {
        "YouTube (Browser)": "https://www.youtube.com",
        "Chrome": r"C:\Program Files\Google\Chrome\Application\chrome.exe"},
    
    "Text Editors": {
        "Notepad": r"C:\Windows\System32\notepad.exe",
        "VS Code": r"C:\Program Files\Microsoft VS Code\Code.exe",},
    
    "Office": {
        "PowerPoint": r"C:\Program Files\Microsoft Office\root\Office16\POWERPNT.EXE",
        "MS Word": r"C:\Program Files\Microsoft Office\root\Office16\WINWORD.EXE",
        "MS Excel": r"C:\Program Files\Microsoft Office\root\Office16\EXCEL.EXE",  },
    
    "Utilities": {
        "Calculator": r"C:\Windows\System32\calc.exe",
        "Paint": r"C:\Windows\System32\mspaint.exe", },
    
    "Custom App": {
        "Custom...": None }
}

selected_app_path = None

# Webcam



     cam = cv2.VideoCapture(0)

    def get_eye_distance(landmarks, top_idx, bottom_idx):
    top = landmarks[top_idx]
    bottom = landmarks[bottom_idx]
    return abs(top.y - bottom.y)

    def smooth_move(current_x, current_y, target_x, target_y, factor=0.35):
    new_x = current_x + (target_x - current_x) * factor
    new_y = current_y + (target_y - current_y) * factor
    return new_x, new_y

    def play_click_sound():
    try:
        click_sound.play()  # Play the sound using pygame mixer
    except Exception as e:
        print("Sound play error:", e)

    def save_calibration(threshold):
    with open("config.txt", "w") as f:
        f.write(str(threshold))

    def load_calibration():
    global blink_threshold, calibrated
    if os.path.exists("config.txt"):
        with open("config.txt", "r") as f:
            blink_threshold = float(f.read().strip())
            calibrated = True
            messagebox.showinfo("Calibration Loaded", f"Threshold loaded: {blink_threshold:.5f}")

    def save_selected_app(path):
    with open(APP_CONFIG_PATH, 'w') as f:
        json.dump({"selected": path}, f)

    def load_selected_app():
    global selected_app_path
    if os.path.exists(APP_CONFIG_PATH):
        with open(APP_CONFIG_PATH, 'r') as f:
            data = json.load(f)
            selected_app_path = data.get("selected")

    def calibrate_blink():
    global blink_threshold, calibrated
    messagebox.showinfo("Calibration", "Keep your eyes OPEN and press OK.")
    open_eye_distances = []
    for _ in range(20):
        _, frame = cam.read()
        frame = cv2.flip(frame, 1)
        rgb = cv2.cvtColor(frame, cv2.COLOR_BGR2RGB)
        results = face_mesh.process(rgb)
        if results.multi_face_landmarks:
            lm = results.multi_face_landmarks[0].landmark
            open_eye_distances.append(
                (get_eye_distance(lm, 159, 145) + get_eye_distance(lm, 386, 374)) / 2
            )
        cv2.waitKey(30)

    messagebox.showinfo("Calibration", "Now CLOSE your eyes and press OK.")
    closed_eye_distances = []
    for _ in range(20):
        _, frame = cam.read()
        frame = cv2.flip(frame, 1)
        rgb = cv2.cvtColor(frame, cv2.COLOR_BGR2RGB)
        results = face_mesh.process(rgb)
        if results.multi_face_landmarks:
            lm = results.multi_face_landmarks[0].landmark
            closed_eye_distances.append(
                (get_eye_distance(lm, 159, 145) + get_eye_distance(lm, 386, 374)) / 2
            )
        cv2.waitKey(30)

    avg_open = np.mean(open_eye_distances)
    avg_closed = np.mean(closed_eye_distances)
    blink_threshold = (avg_open + avg_closed) / 2 * 0.85
    calibrated = True
    save_calibration(blink_threshold)
    messagebox.showinfo("Done", f"Calibration complete.\nThreshold set to {blink_threshold:.5f}")

    def open_app():
    global selected_app_path
    if selected_app_path:
        try:
            print(f"Attempting to open: {selected_app_path}")  # Debug output
            if selected_app_path.startswith("http"):
                # For URL, it should open in the default browser
                subprocess.Popen(["start", "", selected_app_path], shell=True)
            else:
                # For local apps, make sure to pass the correct path
                subprocess.Popen(selected_app_path, shell=True)
            print(f"Successfully launched {selected_app_path}")
        except Exception as e:
            print(f"Error opening app: {e}")

     def on_app_select(event):
    global selected_app_path
    selection = selected_app.get()
    for category in APPS:
        if selection in APPS[category]:
            if APPS[category][selection] is None:
                selected_app_path = filedialog.askopenfilename(title="Select Application")
            else:
                selected_app_path = APPS[category][selection]
            save_selected_app(selected_app_path)
            break

    def track_face():
    global prev_mouse_x, prev_mouse_y, last_click_time, blink_start_time, last_blink_time, blink_count

    while True:
        if not tracking_enabled or not calibrated:
            time.sleep(0.1)
            continue

        ret, frame = cam.read()
        if not ret:
            break

        frame = cv2.flip(frame, 1)
        rgb = cv2.cvtColor(frame, cv2.COLOR_BGR2RGB)
        h, w, _ = frame.shape
        results = face_mesh.process(rgb)

        if results.multi_face_landmarks:
            lm = results.multi_face_landmarks[0].landmark

            # Iris tracking
            iris_x = np.mean([lm[i].x for i in [474, 475, 476, 477]])
            iris_y = np.mean([lm[i].y for i in [474, 475, 476, 477]])
            target_x = iris_x * screen_w
            target_y = iris_y * screen_h
            prev_mouse_x, prev_mouse_y = smooth_move(prev_mouse_x, prev_mouse_y, target_x, target_y)
            pyautogui.moveTo(prev_mouse_x, prev_mouse_y)

            # Scroll gestures
            if iris_y < 0.35:
                pyautogui.scroll(10)
            elif iris_y > 0.65:
                pyautogui.scroll(-10)

            # Eye blink detection
            left_eye = get_eye_distance(lm, 159, 145)
            right_eye = get_eye_distance(lm, 386, 374)

            eye_history_left.append(left_eye)
            if len(eye_history_left) > eye_history_length:
                eye_history_left.pop(0)
            avg_left = np.mean(eye_history_left)

            eye_history_right.append(right_eye)
            if len(eye_history_right) > eye_history_length:
                eye_history_right.pop(0)
            avg_right = np.mean(eye_history_right)

            current_time = time.time()
            is_left_closed = avg_left < blink_threshold
            is_right_closed = avg_right < blink_threshold

            if is_left_closed and is_right_closed:
                if blink_start_time is None:
                    blink_start_time = current_time
            else:
                if blink_start_time:
                    blink_duration = current_time - blink_start_time
                    blink_start_time = None
                    if 0.1 < blink_duration < 0.5:
                        if current_time - last_blink_time < double_blink_window:
                            blink_count += 1
                        else:
                            blink_count = 1
                        last_blink_time = current_time

                        if blink_count == 2:
                            open_app()
                            play_click_sound()
                            blink_count = 0
                        elif current_time - last_click_time > click_cooldown:
                            pyautogui.click()
                            play_click_sound()
                            last_click_time = current_time

            # Wink detection
            if is_left_closed and not is_right_closed:
                pyautogui.hotkey('ctrl', 'left')
            elif is_right_closed and not is_left_closed:
                pyautogui.hotkey('ctrl', 'right')

            if overlay_enabled:
                for idx in [145, 159, 374, 386, 474, 475, 476, 477]:
                    x = int(lm[idx].x * w)
                    y = int(lm[idx].y * h)
                    cv2.circle(frame, (x, y), 2, (0, 255, 0), -1)
                cx = int(target_x / screen_w * w)
                cy = int(target_y / screen_h * h)
                cv2.drawMarker(frame, (cx, cy), (255, 0, 0), cv2.MARKER_CROSS, 20, 2)

        if overlay_enabled:
            cv2.imshow("Eye Mouse Overlay", frame)
            if cv2.waitKey(1) & 0xFF == 27:
                break

    def start_tracking():
    global tracking_enabled
    if not calibrated:
        messagebox.showwarning("Warning", "Please calibrate first.")
        return
    tracking_enabled = True
    start_btn.config(state="disabled")
    stop_btn.config(state="normal")

    def stop_tracking():
    global tracking_enabled
    tracking_enabled = False
    start_btn.config(state="normal")
    stop_btn.config(state="disabled")

    def toggle_overlay():
    global overlay_enabled
    overlay_enabled = not overlay_enabled
    overlay_btn.config(text=f"Overlay: {'ON' if overlay_enabled else 'OFF'}")

# GUI Setup
      root = tk.Tk()
     root.title("Eye Mouse Control")
    selected_app = tk.StringVar()

    frame = tk.Frame(root, padx=10, pady=10)
    frame.pack()

    tk.Label(frame, text="Eye Mouse - Enhanced", font=("Arial", 14, "bold")).pack(pady=10)

    start_btn = tk.Button(frame, text="Start Tracking", command=start_tracking)
     start_btn.pack(pady=5)

     stop_btn = tk.Button(frame, text="Stop Tracking", command=stop_tracking, state="disabled")
    stop_btn.pack(pady=5)

    calibrate_btn = tk.Button(frame, text="Calibrate Blink", command=calibrate_blink)
    calibrate_btn.pack(pady=5)

    load_btn = tk.Button(frame, text="Load Calibration", command=load_calibration)
    load_btn.pack(pady=5)

    overlay_btn = tk.Button(frame, text="Overlay: ON", command=toggle_overlay)
    overlay_btn.pack(pady=5)

# App Dropdown

     tk.Label(frame, text="Double Blink App").pack(pady=(10, 0))
    app_menu = ttk.Combobox(frame, textvariable=selected_app)
    app_menu['values'] = [f"{app}" for group in APPS.values() for app in group.keys()]
    app_menu.bind("<<ComboboxSelected>>", on_app_select)
    app_menu.pack(pady=5)

    exit_btn = tk.Button(frame, text="Exit", command=root.destroy)
    exit_btn.pack(pady=10)

    load_selected_app()
    threading.Thread(target=track_face, daemon=True).start()
    root.mainloop()

# Cleanup
    cam.release()
    cv2.destroyAllWindows()
