import os
from pathlib import Path
import pytesseract
from PIL import Image
from langdetect import detect
import docx
import fitz
import pdf2image
import re
import sys
from googletrans import Translator
from typing import Tuple

class DocumentProcessor:
    def __init__(self):
        if sys.platform.startswith('win'):
            os.environ['PYTHONIOENCODING'] = 'utf-8'
        self.setup_directories()
        self.supported_formats = {'.pdf', '.jpg', '.jpeg', '.png'}
        self.translator = Translator()

    def setup_directories(self):
        self.input_dir = Path("Input")
        self.output_dir = Path("Output")
        self.input_dir.mkdir(exist_ok=True)
        self.output_dir.mkdir(exist_ok=True)
        print("Utworzono katalogi Input i Output")
        print("\nProgram obsługuje formaty: PDF, JPG, JPEG, PNG")
        print("Umieść pliki w folderze Input i naciśnij ENTER")
        input()

    def process_image(self, image_path):
        image = Image.open(image_path)
        text = pytesseract.image_to_string(image, lang='eng')
        return text.strip()

    def process_pdf(self, pdf_path):
        doc = fitz.open(pdf_path)
        text = "".join(page.get_text() for page in doc)
        doc.close()
        return text.strip()

    def detect_and_process_text(self, text: str) -> Tuple[str, str]:
        """
        Detect language and process text according to language rules.
        Returns tuple of (processed_text, detected_language)
        """
        try:
            detected_lang = detect(text)
            
            if detected_lang == 'pl':
                # For Polish text, improve spelling using translation back and forth
                improved = self.translator.translate(text, dest='en')
                final_text = self.translator.translate(improved.text, dest='pl').text
            else:
                # For non-Polish text, translate to Polish
                final_text = self.translator.translate(text, dest='pl').text
            
            return final_text, detected_lang
            
        except Exception as e:
            print(f"Warning: Language processing error: {e}")
            return text, 'unknown'

    def save_document(self, text, output_path, format_type):
        if format_type == 'docx':
            doc = docx.Document()
            doc.add_paragraph(text)
            doc.save(output_path)
        else:
            with open(output_path, 'w', encoding='utf-8') as f:
                f.write(text)

    def process_documents(self):
        files = [f for f in self.input_dir.glob('*') if f.suffix.lower() in self.supported_formats]
        if not files:
            print("Brak plików do przetworzenia w folderze Input")
            return

        print(f"\nZnaleziono {len(files)} plików do przetworzenia")
        
        if len(files) > 1:
            merge_option = input("Wykryto wiele plików. Czy chcesz połączyć je w jeden plik wyjściowy? (t/n): ").lower()
            
            if merge_option == 't':
                all_text = []
                for file_path in files:
                    print(f"\nPrzetwarzanie: {file_path.name}")
                    if file_path.suffix.lower() == '.pdf':
                        text = self.process_pdf(file_path)
                    else:
                        text = self.process_image(file_path)
                    
                    processed_text, lang = self.detect_and_process_text(text)
                    print(f"Wykryty język: {lang}")
                    all_text.append(processed_text)
                
                format_type = ''
                while format_type not in ['docx', 'txt']:
                    format_type = input("Wybierz format wyjściowy (docx/txt): ").lower()
                
                output_name = input(f"Podaj nazwę pliku wyjściowego (bez .{format_type}): ")
                output_path = self.output_dir / f"{output_name}.{format_type}"
                
                self.save_document('\n\n'.join(all_text), output_path, format_type)
                print(f"Zapisano: {output_name}.{format_type}")
                
            else:
                for file_path in files:
                    print(f"\nPrzetwarzanie: {file_path.name}")
                    if file_path.suffix.lower() == '.pdf':
                        text = self.process_pdf(file_path)
                    else:
                        text = self.process_image(file_path)
                    
                    processed_text, lang = self.detect_and_process_text(text)
                    print(f"Wykryty język: {lang}")
                    
                    format_type = ''
                    while format_type not in ['docx', 'txt']:
                        format_type = input("Wybierz format wyjściowy (docx/txt): ").lower()
                    
                    output_name = input(f"Podaj nazwę pliku wyjściowego (bez .{format_type}): ")
                    output_path = self.output_dir / f"{output_name}.{format_type}"
                    
                    self.save_document(processed_text, output_path, format_type)
                    print(f"Zapisano: {output_name}.{format_type}")
        else:
            file_path = files[0]
            print(f"\nPrzetwarzanie: {file_path.name}")
            
            if file_path.suffix.lower() == '.pdf':
                text = self.process_pdf(file_path)
            else:
                text = self.process_image(file_path)
            
            processed_text, lang = self.detect_and_process_text(text)
            print(f"Wykryty język: {lang}")
            
            format_type = ''
            while format_type not in ['docx', 'txt']:
                format_type = input("Wybierz format wyjściowy (docx/txt): ").lower()
            
            output_name = input(f"Podaj nazwę pliku wyjściowego (bez .{format_type}): ")
            output_path = self.output_dir / f"{output_name}.{format_type}"
            
            self.save_document(processed_text, output_path, format_type)
            print(f"Zapisano: {output_name}.{format_type}")

if __name__ == "__main__":
    DocumentProcessor().process_documents()