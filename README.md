current_language = 'C'  # Default language

def get_tkinter_index(text_widget, char_index):
    line = text_widget.index(f"1.0+{char_index}c").split(".")
    return f"{line[0]}.{line[1]}"

def highlight_code(text_widget, code):
    language = LANGUAGES[current_language]
    patterns = language['patterns']
    
    # Remove all tags first
    for token in TOKEN_TYPES.keys():
        text_widget.tag_remove(token, "1.0", END)
    
    # Apply highlighting for each token type
    for token_type, pattern in patterns.items():
        if token_type not in TOKEN_TYPES:
            continue
            
        for match in re.finditer(pattern, code, re.MULTILINE):
            start, end = match.span()
            start_index = get_tkinter_index(text_widget, start)
            end_index = get_tkinter_index(text_widget, end)
            text_widget.tag_add(token_type, start_index, end_index)
            text_widget.tag_configure(token_type, foreground=TOKEN_TYPES[token_type])
    
    return code

def run(code):
    terminal_output.config(state=NORMAL)
    terminal_output.delete("1.0", END)
    terminal_output.config(state=DISABLED)

    if not code.strip():
        return

    language = LANGUAGES[current_language]
    filename = f"temp{language['extension']}"
    executable = "a.exe" if current_language in ['C', 'C++'] else "Main.class" if current_language == 'Java' else filename
    
    try:
        with open(filename, "w", encoding='utf-8') as fp:
            fp.write(code)

        if current_language == 'Python':
            # Python is interpreted, no compilation needed
            try:
                run_process = subprocess.run(
                    [language['interpreter'], filename],
                    stdout=subprocess.PIPE,
                    stderr=subprocess.PIPE,
                    text=True,
                    timeout=10,
                    shell=True  # Add shell=True for Windows
                )
                
                terminal_output.config(state=NORMAL)
                terminal_output.insert(END, "Output:\n" + run_process.stdout)
                if run_process.stderr:
                    terminal_output.insert(END, "\nErrors:\n" + run_process.stderr)
                terminal_output.config(state=DISABLED)
                
            except subprocess.TimeoutExpired:
                terminal_output.config(state=NORMAL)
                terminal_output.insert(END, "Execution timed out.")
                terminal_output.config(state=DISABLED)
            except Exception as e:
                terminal_output.config(state=NORMAL)
                terminal_output.insert(END, f"Error: {str(e)}")
                terminal_output.config(state=DISABLED)
                
        else:
            # Compiled languages (C, C++, Java)
            compiler = language['compiler']
            
            # Java needs special handling
            if current_language == 'Java':
                compile_process = subprocess.run(
                    [compiler, filename],
                    stdout=subprocess.PIPE,
                    stderr=subprocess.PIPE,
                    text=True,
                    shell=True
                )
                run_command = ['java', 'Main']
            else:
                compile_process = subprocess.run(
                    [compiler, filename, '-o', executable],
                    stdout=subprocess.PIPE,
                    stderr=subprocess.PIPE,
                    text=True,
                    shell=True
                )
                run_command = [executable]

            terminal_output.config(state=NORMAL)
            if compile_process.returncode != 0:
                terminal_output.insert(END, "Compilation Error:\n" + compile_process.stderr)
                terminal_output.config(state=DISABLED)
                return

            try:
                run_process = subprocess.run(
                    run_command,
                    stdout=subprocess.PIPE,
                    stderr=subprocess.PIPE,
                    text=True,
                    timeout=10,
                    shell=True
                )
                
                terminal_output.insert(END, "Output:\n" + run_process.stdout)
                if run_process.stderr:
                    terminal_output.insert(END, "\nErrors:\n" + run_process.stderr)
                    
            except subprocess.TimeoutExpired:
                terminal_output.insert(END, "Execution timed out.")
            except Exception as e:
                terminal_output.insert(END, f"Error: {str(e)}")
                
            terminal_output.config(state=DISABLED)

    except Exception as e:
        terminal_output.config(state=NORMAL)
        terminal_output.insert(END, f"Error: {str(e)}")
        terminal_output.config(state=DISABLED)

def detect_errors(code):
    highlight_code(text_area, code)
    text_area.tag_remove("error_line", "1.0", END)
    terminal_output.config(state=NORMAL)
    terminal_output.delete("1.0", END)

    language = LANGUAGES[current_language]
    filename = f"temp_live{language['extension']}"
    
    with open(filename, "w") as f:
        f.write(code)

    if current_language == 'Python':
        # Python syntax checking
        process = subprocess.run(
            [language['interpreter'], "-m", "py_compile", filename],
            stdout=subprocess.PIPE,
            stderr=subprocess.PIPE,
            text=True
        )
    else:
        # Compiled languages syntax checking
        compiler = language['compiler']
        compile_command = [compiler, "-fsyntax-only", filename]
        if current_language == 'Java':
            compile_command = [compiler, "-Xlint", filename]
            
        process = subprocess.run(
            compile_command,
            stdout=subprocess.PIPE,
            stderr=subprocess.PIPE,
            text=True
        )

    # Display and highlight errors
    if process.stderr:
        error_lines = set()
        for line in process.stderr.splitlines():
            if current_language == 'Python':
                match = re.search(r'File "' + re.escape(filename) + r'", line (\d+)', line)
            else:
                match = re.search(r"" + re.escape(filename) + r":(\d+):", line)
                
            if match:
                error_lines.add(int(match.group(1)))

        terminal_output.insert(END, process.stderr)

        for line_num in error_lines:
            index_start = f"{line_num}.0"
            index_end = f"{line_num}.end"
            text_area.tag_add("error_line", index_start, index_end)

        text_area.tag_configure("error_line", background="#FF5555") 
    else:
        terminal_output.insert(END, "No syntax errors detected.")

    terminal_output.config(state=DISABLED)

def compute_lps(pattern):
    lps = [0] * len(pattern)
    length = 0
    i = 1
    while i < len(pattern):
        if pattern[i] == pattern[length]:
            length += 1
            lps[i] = length
            i += 1
        else:
            if length != 0:
                length = lps[length - 1]
            else:
                lps[i] = 0
                i += 1
    return lps

def kmp_prefix_match(text, pattern):
    lps = compute_lps(pattern)
    i = j = 0
    while i < len(text) and j < len(pattern):
        if text[i] == pattern[j]:
            i += 1
            j += 1
        elif j != 0:
            j = lps[j - 1]
        else:
            i += 1
    return j == len(pattern)

def autocomplete_kmp(prefix, keyword_list):
    return [word for word in keyword_list if kmp_prefix_match(word, prefix)]

def update_line_numbers(event=None):
    line_count_widget.config(state=NORMAL)
    line_count_widget.delete(1.0, END)
    lines = text_area.index('end-1c').split('.')[0]
    line_numbers = "\n".join(str(i) for i in range(1, int(lines) + 1))
    line_count_widget.insert(END, line_numbers)
    line_count_widget.config(state=DISABLED)
    scroll_line_numbers()

def sync_scroll(*args):
    text_area.yview(*args)
    line_count_widget.yview(*args)

def on_text_scroll(event):
    text_area.yview_scroll(-1 * (event.delta // 120), "units")
    line_count_widget.yview_scroll(-1 * (event.delta // 120), "units")

def scroll_line_numbers(*args):
    line_count_widget.yview_moveto(text_area.yview()[0])

def on_cursor_move(event=None):
    cursor_line = int(text_area.index(INSERT).split('.')[0])
    visible_first, visible_last = text_area.yview()
    total_lines = int(text_area.index('end-1c').split('.')[0])
    first_visible_line = int(visible_first * total_lines)
    last_visible_line = int(visible_last * total_lines)

    if cursor_line > last_visible_line - 3:
        text_area.yview_scroll(1, "units")
        line_count_widget.yview_scroll(1, "units")
    elif cursor_line < first_visible_line + 3:
        text_area.yview_scroll(-1, "units")
        line_count_widget.yview_scroll(-1, "units")

current_file = None  # Global variable to store current file path

def new_file():
    global current_file
    file_path = filedialog.asksaveasfilename(
        defaultextension=LANGUAGES[current_language]['extension'],
        filetypes=[("All Files", "*.*")] + 
                  [(f"{lang} Files", f"*{LANGUAGES[lang]['extension']}") for lang in LANGUAGES],
        title="Create New File"
    )
    if file_path:
        with open(file_path, "w") as f:
            f.write("")  
        current_file = file_path
        text_area.delete("1.0", END)

def open_file():
    global current_file
    file_path = filedialog.askopenfilename(
        filetypes=[("All Files", "*.*")] + 
                  [(f"{lang} Files", f"*{LANGUAGES[lang]['extension']}") for lang in LANGUAGES]
    )
    if file_path:
        # Detect language based on file extension
        ext = os.path.splitext(file_path)[1].lower()
        for lang, config in LANGUAGES.items():
            if ext == config['extension']:
                set_language(lang)
                break
                
        with open(file_path, "r") as f:
            text_area.delete("1.0", END)
            text_area.insert("1.0", f.read())
        current_file = file_path

def save_file():
    global current_file
    if current_file:
        with open(current_file, "w") as f:
            f.write(text_area.get("1.0", END))
    else:
        save_as_file()

def save_as_file():
    global current_file
    file_path = filedialog.asksaveasfilename(
        defaultextension=LANGUAGES[current_language]['extension'],
        filetypes=[("All Files", "*.*")] + 
                  [(f"{lang} Files", f"*{LANGUAGES[lang]['extension']}") for lang in LANGUAGES]
    )
    if file_path:
        with open(file_path, "w") as f:
            f.write(text_area.get("1.0", END))
        current_file = file_path

def set_language(lang):
    global current_language
    current_language = lang
    language_label.config(text=f"Language: {lang}")
    # Re-highlight the code with new language rules
    highlight_code(text_area, text_area.get("1.0", END))

def show_autocomplete():
    cursor_index = text_area.index(INSERT)
    line, col = map(int, cursor_index.split('.'))
    line_text = text_area.get(f"{line}.0", f"{line}.end")
    match = re.search(r'\w*$', line_text[:col])
    if not match:
        hide_autocomplete()
        return
    prefix = match.group(0)

    language = LANGUAGES[current_language]
    
    if line_text.strip().startswith("#include"):
        suggestions = autocomplete_kmp(prefix, language.get('headers', []))
    elif current_language == 'Python' and line_text.strip().startswith('import'):
        suggestions = autocomplete_kmp(prefix, language.get('modules', []))
    else:
        suggestions = autocomplete_kmp(prefix, 
            language.get('keywords', []) + 
            language.get('functions', []) + 
            language.get('classes', [])
        )

    if not prefix or not prefix.isalpha():
        hide_autocomplete()
        return

    if not suggestions:
        hide_autocomplete()
        return

    autocomplete_listbox.delete(0, END)
    for word in suggestions:
        autocomplete_listbox.insert(END, word)

    bbox = text_area.bbox(cursor_index)
    if not bbox:
        hide_autocomplete()
        return

    x, y, _, height = bbox
    abs_x = text_area.winfo_rootx() + x
    abs_y = text_area.winfo_rooty() + y + height
    autocomplete_listbox.place(x=abs_x, y=abs_y)
    autocomplete_listbox.lift()

# GUI Setup
root = Tk()
screen_width = root.winfo_screenwidth()
screen_height = root.winfo_screenheight()
root.resizable(True, True)
root.geometry(f"{screen_width}x{screen_height}")
root.title("Multi-Language Code Editor")

root.configure(bg='#1e1e1e')
font_config = ('Fira Code', 20)

# Language selection dropdown
language_var = StringVar(value=current_language)
language_menu = OptionMenu(root, language_var, *LANGUAGES.keys(), command=set_language)
language_menu.config(bg='#44475a', fg='#f8f8f2', activebackground='#6272a4', 
                    activeforeground='#f8f8f2', highlightthickness=0)
language_menu.place(x=100, y=5)

language_label = Label(root, text=f"Language: {current_language}", bg='#1e1e1e', fg='#f8f8f2')
language_label.place(x=250, y=10)

# Autocomplete listbox
autocomplete_listbox = Listbox(root, height=5, bg='#1e1e1e', fg='#f8f8f2',
                               font=('Fira Code', 14), highlightthickness=0, selectbackground='#6272a4')
autocomplete_listbox.place_forget()

def hide_autocomplete():
    autocomplete_listbox.place_forget()

def insert_autocomplete_selection(event=None):
    if autocomplete_listbox.size() == 0:
        return
    selected = autocomplete_listbox.get(ACTIVE)
    cursor_index = text_area.index(INSERT)
    line, col = map(int, cursor_index.split('.'))
    line_text = text_area.get(f"{line}.0", f"{line}.end")
    match = re.search(r'\w*$', line_text[:col])
    if match:
        start_col = match.start()
        text_area.delete(f"{line}.{start_col}", f"{line}.{col}")
        text_area.insert(f"{line}.{start_col}", selected)
    hide_autocomplete()
    text_area.focus_set()
    return "break"

autocomplete_listbox.bind("<Return>", insert_autocomplete_selection)
autocomplete_listbox.bind("<ButtonRelease-1>", insert_autocomplete_selection)

# File menu
file_menu = Menu(root, tearoff=0, bg='#44475a', fg='#f8f8f2', 
                activebackground='#6272a4', activeforeground='#f8f8f2', 
                font=('Fira Code', 10))
file_menu.add_command(label="New File", command=new_file)
file_menu.add_command(label="Open", command=open_file)
file_menu.add_command(label="Save", command=save_file)
file_menu.add_command(label="Save As", command=save_as_file)

def show_file_menu(event):
    try:
        file_menu.tk_popup(event.x_root, event.y_root)
    finally:
        file_button.configure(bg='#50fa7b')

# Scrollbar style
style = ttk.Style()
style.theme_use('clam')
style.configure("TScrollbar", gripcount=0,
                background="#44475a", darkcolor="#44475a", lightcolor="#44475a",
                troughcolor="#282a36", bordercolor="#282a36", arrowcolor="#f8f8f2")

# Main scrollbar
scrollbar = ttk.Scrollbar(root, command=sync_scroll)
scrollbar.place(x=screen_width - 40, y=40, height=screen_height - 250)

# Editor frame
frame = Frame(root, bg='#282a36')
frame.place(x=10, y=40, width=screen_width - 80, height=screen_height - 250)

# Line numbers
line_count_widget = Text(frame, width=5, padx=10, pady=10, bg='#1e1e1e', fg='#6272a4',
                         insertbackground='white', font=font_config, state=DISABLED)
line_count_widget.pack(side=LEFT, fill=Y)

# Text editor
text_area = Text(frame, padx=10, pady=10, bg='#1e1e1e', fg='#f8f8f2', insertbackground='white',
                 selectbackground='#44475a', font=font_config, wrap="none",
                 yscrollcommand=lambda *args: (scrollbar.set(*args), scroll_line_numbers(*args)))
text_area.pack(side=RIGHT, fill=BOTH, expand=True)

# Buttons
file_button = Button(root, text="FILE", padx=20, pady=2, bg='#50fa7b', fg='#282a36', 
                    activebackground='#8be9fd', activeforeground='#282a36', 
                    borderwidth=0, font=('Helvetica', 12, 'bold'))
file_button.place(x=15, y=5)

Button(root, text="RUN", padx=20, pady=2, bg='#50fa7b', fg='#282a36', 
       activebackground='#8be9fd', activeforeground='#282a36', 
       borderwidth=0, font=('Helvetica', 12, 'bold'),
       command=lambda: run(text_area.get("1.0", END))).place(x=screen_width - 160, y=5)

# Terminal output
terminal_output = Text(root, height=6, bg='#1e1e1e', fg='#f8f8f2', insertbackground='white',
                       font=('Courier', 12), wrap="word")
terminal_output.place(x=10, y=screen_height - 200, width=screen_width - 80, height=150)
terminal_output.config(state=DISABLED)

terminal_scroll = ttk.Scrollbar(root, command=terminal_output.yview)
terminal_scroll.place(x=screen_width - 40, y=screen_height - 200, height=150)
terminal_output.config(yscrollcommand=terminal_scroll.set)

# Bindings
def focus_autocomplete_if_visible(event):
    if autocomplete_listbox.winfo_ismapped():
        autocomplete_listbox.focus_set()
        autocomplete_listbox.selection_set(0)
        return "break"

text_area.bind("<Down>", focus_autocomplete_if_visible)
text_area.bind("<Control-space>", lambda e: show_autocomplete())

text_area.config(yscrollcommand=sync_scroll)
text_area.tag_configure("error_line", underline=True, background="#FF5555")
text_area.bind("<KeyRelease>", lambda e: (update_line_numbers(), detect_errors(text_area.get("1.0", END)), show_autocomplete()))

line_count_widget.config(yscrollcommand=sync_scroll)
text_area.bind("<MouseWheel>", on_text_scroll)
line_count_widget.bind("<MouseWheel>", on_text_scroll)
text_area.bind("<KeyPress>", on_cursor_move)
file_button.bind("<Button-1>", show_file_menu)
text_area.bind("<Escape>", lambda e: hide_autocomplete())

# Initialize
update_line_numbers()
root.mainloop()
