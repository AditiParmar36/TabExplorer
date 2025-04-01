

import sys
import os
import mimetypes
from PyQt5.QtWidgets import (
    QApplication, QMainWindow, QFileSystemModel, QTreeView, QListWidget, QListWidgetItem,
    QVBoxLayout, QWidget, QSplitter, QLabel, QTextEdit, QPushButton, QHBoxLayout, QComboBox
)
from PyQt5.QtGui import QPixmap
from PyQt5.QtCore import Qt, QDir
import fitz  # PyMuPDF for PDF rendering

class FileExplorer(QMainWindow):
    def __init__(self):
        super().__init__()
        self.setWindowTitle("File Explorer with Theme & Auto-Categorization")
        self.setGeometry(100, 100, 1200, 700)
        self.current_theme = "dark"  # start with dark theme
        self.auto_categorize_enabled = True  # start with auto-categorization enabled
        self.current_dir = None  # store current directory for refreshing file list
        self.initUI()
        
    def initUI(self):
        # Main widget and layout
        self.central_widget = QWidget()
        self.setCentralWidget(self.central_widget)
        self.main_layout = QVBoxLayout(self.central_widget)
        
        # Top layout with Theme Toggle and Auto Categorization Toggle Buttons
        self.top_layout = QHBoxLayout()
        self.theme_button = QPushButton("Switch to Light Theme")
        self.theme_button.clicked.connect(self.toggleTheme)
        self.top_layout.addWidget(self.theme_button)
        
        self.auto_cat_button = QPushButton("Auto-Categorization: ON")
        self.auto_cat_button.clicked.connect(self.toggleAutoCategorization)
        self.top_layout.addWidget(self.auto_cat_button)
        
        self.top_layout.addStretch()  # push buttons to left
        self.main_layout.addLayout(self.top_layout)
        
        # Main splitter: left for drive selector and directory tree, right for file list and preview
        self.splitter = QSplitter(Qt.Horizontal)
        self.main_layout.addWidget(self.splitter)
        
        # LEFT SIDE: A widget with vertical layout for drive selection and directory tree.
        self.left_widget = QWidget()
        self.left_layout = QVBoxLayout(self.left_widget)
        
        # Drive Selector ComboBox
        self.drive_combo = QComboBox()
        self.populateDrives()
        self.drive_combo.currentIndexChanged.connect(self.onDriveChanged)
        self.left_layout.addWidget(self.drive_combo)
        
        # Directory Tree using QFileSystemModel & QTreeView
        self.dir_model = QFileSystemModel()
        # Initially set to the first drive
        initial_drive = self.drive_combo.currentText()
        self.dir_model.setRootPath(initial_drive)
        
        self.tree = QTreeView()
        self.tree.setModel(self.dir_model)
        self.tree.setRootIndex(self.dir_model.index(initial_drive))
        self.tree.setHeaderHidden(True)
        self.tree.clicked.connect(self.onDirClicked)
        self.left_layout.addWidget(self.tree)
        
        self.splitter.addWidget(self.left_widget)
        
        # RIGHT SIDE: File list and preview
        self.right_widget = QWidget()
        self.right_layout = QVBoxLayout(self.right_widget)
        
        # File List Widget (displays files in the selected directory)
        self.file_list = QListWidget()
        self.file_list.itemClicked.connect(self.onFileClicked)
        self.right_layout.addWidget(self.file_list)
        
        # Preview Area: a QLabel for images/PDFs and a QTextEdit for text files
        self.preview_label = QLabel("File Preview")
        self.preview_label.setAlignment(Qt.AlignCenter)
        self.preview_label.setMinimumHeight(400)
        self.right_layout.addWidget(self.preview_label)
        
        self.text_preview = QTextEdit()
        self.text_preview.setReadOnly(True)
        self.text_preview.hide()
        self.right_layout.addWidget(self.text_preview)
        
        self.splitter.addWidget(self.right_widget)
        self.splitter.setStretchFactor(1, 1)
        
        # Apply initial dark theme style sheet
        self.applyDarkTheme()
        
    def populateDrives(self):
        self.drive_combo.clear()
        drives = QDir.drives()
        for drive in drives:
            self.drive_combo.addItem(drive.absolutePath())
    
    def onDriveChanged(self, index):
        drive_path = self.drive_combo.itemText(index)
        self.dir_model.setRootPath(drive_path)
        self.tree.setRootIndex(self.dir_model.index(drive_path))
        self.file_list.clear()
        self.preview_label.clear()
        self.preview_label.setText("File Preview")
        self.text_preview.clear()
        self.text_preview.hide()
        self.preview_label.show()
        
    def toggleTheme(self):
        if self.current_theme == "dark":
            self.applyLightTheme()
            self.current_theme = "light"
            self.theme_button.setText("Switch to Dark Theme")
        else:
            self.applyDarkTheme()
            self.current_theme = "dark"
            self.theme_button.setText("Switch to Light Theme")
    
    def toggleAutoCategorization(self):
        self.auto_categorize_enabled = not self.auto_categorize_enabled
        state_text = "ON" if self.auto_categorize_enabled else "OFF"
        self.auto_cat_button.setText(f"Auto-Categorization: {state_text}")
        # Refresh file list if a directory is currently selected
        if self.current_dir:
            self.populateFileList(self.current_dir)
    
    def applyDarkTheme(self):
        dark_style = """
        QWidget {
            background-color: #2b2b2b;
            color: #ffffff;
            font-size: 12px;
        }
        QTreeView, QListWidget, QComboBox {
            background-color: #353535;
        }
        QTreeView::item, QListWidget::item, QComboBox {
            padding: 5px;
        }
        QTreeView::item:selected, QListWidget::item:selected, QComboBox::item:selected {
            background-color: #3d6dad;
        }
        QPushButton {
            background-color: #3d6dad;
            color: #ffffff;
            border: none;
            padding: 5px;
        }
        QPushButton:hover {
            background-color: #5485d6;
        }
        QLabel {
            color: #ffffff;
        }
        QTextEdit {
            background-color: #353535;
            color: #ffffff;
        }
        """
        self.setStyleSheet(dark_style)
    
    def applyLightTheme(self):
        light_style = """
        QWidget {
            background-color: #f0f0f0;
            color: #000000;
            font-size: 12px;
        }
        QTreeView, QListWidget, QComboBox {
            background-color: #ffffff;
        }
        QTreeView::item, QListWidget::item, QComboBox {
            padding: 5px;
        }
        QTreeView::item:selected, QListWidget::item:selected, QComboBox::item:selected {
            background-color: #a0a0ff;
        }
        QPushButton {
            background-color: #a0a0ff;
            color: #000000;
            border: none;
            padding: 5px;
        }
        QPushButton:hover {
            background-color: #c0c0ff;
        }
        QLabel {
            color: #000000;
        }
        QTextEdit {
            background-color: #ffffff;
            color: #000000;
        }
        """
        self.setStyleSheet(light_style)
    
    def onDirClicked(self, index):
        path = self.dir_model.filePath(index)
        if os.path.isdir(path):
            self.current_dir = path
            self.populateFileList(path)
    
    def populateFileList(self, folder):
        self.file_list.clear()
        # Reset preview area
        self.preview_label.clear()
        self.preview_label.setText("File Preview")
        self.text_preview.clear()
        self.text_preview.hide()
        self.preview_label.show()
        
        try:
            if self.auto_categorize_enabled:
                # Group files into categories
                categories = {"Documents": [], "Images": [], "Videos": [], "Others": []}
                for item in os.listdir(folder):
                    full_path = os.path.join(folder, item)
                    if os.path.isfile(full_path):
                        mime_type, _ = mimetypes.guess_type(full_path)
                        if mime_type:
                            if "image" in mime_type:
                                categories["Images"].append(full_path)
                            elif "video" in mime_type:
                                categories["Videos"].append(full_path)
                            elif "text" in mime_type or "pdf" in mime_type:
                                categories["Documents"].append(full_path)
                            else:
                                categories["Others"].append(full_path)
                        else:
                            categories["Others"].append(full_path)
                # Add items to the list with headers
                for cat, files in categories.items():
                    if files:
                        header_item = QListWidgetItem(f"{cat} ({len(files)} files)")
                        # Make header items non-selectable
                        header_item.setFlags(Qt.ItemIsEnabled)
                        self.file_list.addItem(header_item)
                        for file_path in files:
                            base_name = os.path.basename(file_path)
                            file_item = QListWidgetItem("   " + base_name)
                            file_item.setData(Qt.UserRole, file_path)
                            self.file_list.addItem(file_item)
            else:
                # Simple flat file list
                for item in os.listdir(folder):
                    full_path = os.path.join(folder, item)
                    if os.path.isfile(full_path):
                        file_item = QListWidgetItem(item)
                        file_item.setData(Qt.UserRole, full_path)
                        self.file_list.addItem(file_item)
        except Exception as e:
            print("Error listing files:", e)
    
    def onFileClicked(self, item):
        # If auto categorization is enabled, header items won't have associated file paths.
        file_path = item.data(Qt.UserRole)
        if not file_path or not os.path.exists(file_path):
            return
        
        mime_type, _ = mimetypes.guess_type(file_path)
        if mime_type and "image" in mime_type:
            pixmap = QPixmap(file_path)
            self.preview_label.setPixmap(pixmap.scaled(500, 500, Qt.KeepAspectRatio, Qt.SmoothTransformation))
            self.preview_label.show()
            self.text_preview.hide()
        elif mime_type and "pdf" in mime_type:
            try:
                doc = fitz.open(file_path)
                page = doc.load_page(0)  # load first page
                pix = page.get_pixmap()
                img_data = pix.tobytes("png")
                pdf_pix = QPixmap()
                pdf_pix.loadFromData(img_data)
                self.preview_label.setPixmap(pdf_pix.scaled(500, 500, Qt.KeepAspectRatio, Qt.SmoothTransformation))
                self.preview_label.show()
                self.text_preview.hide()
            except Exception as e:
                self.preview_label.setText("Error loading PDF preview.")
                self.preview_label.show()
                self.text_preview.hide()
        elif mime_type and "text" in mime_type:
            try:
                with open(file_path, "r", encoding="utf-8", errors="ignore") as f:
                    content = f.read()
                self.text_preview.setPlainText(content)
                self.text_preview.show()
                self.preview_label.hide()
            except Exception as e:
                self.preview_label.setText("Error loading text file.")
                self.preview_label.show()
                self.text_preview.hide()
        else:
            self.preview_label.setText("Preview not available for this file type.")
            self.preview_label.show()
            self.text_preview.hide()

if __name__ == "__main__":
    app = QApplication(sys.argv)
    window = FileExplorer()
    window.show()
    sys.exit(app.exec_())
