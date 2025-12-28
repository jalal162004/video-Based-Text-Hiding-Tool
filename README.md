from tkinter import *
import tkinter.filedialog
from tkinter import messagebox
import os
import cv2
from PIL import Image, ImageTk

MAGIC = b"JVID"  # 4 bytes

class jalalvid:
    def main(self, root):
        self.root = root
        root.title("jalalvid")
        root.geometry("520x650")
        root.resizable(False, False)

        f = Frame(root)
        Label(f, text="Video Steganography", font=("courier", 26)).grid(row=0, pady=20)

        Button(f, text="Encode", padx=14, font=("courier", 14),
               command=lambda: self.frame1_encode(f)).grid(row=1, pady=12)

        Button(f, text="Decode", padx=14, font=("courier", 14),
               command=lambda: self.frame1_decode(f)).grid(row=2, pady=12)

        Label(f, text="\n Data Hiding\nIn Video\n", font=("courier", 40)).grid(row=3, pady=30)
        f.grid()

    def home(self, frame):
        frame.destroy()
        self.main(self.root)

    # ---------------- GUI: ENCODE ----------------
    def frame1_encode(self, f):
        f.destroy()
        f2 = Frame(self.root)

        Label(f2, text="ENCODE", font=("courier", 60)).grid(row=0, pady=30)
        Label(f2, text="Select the Video in which\nyou want to hide text:",
              font=("courier", 16)).grid(row=1, pady=10)

        Button(f2, text="Select", font=("courier", 16),
               command=lambda: self.frame2_encode(f2)).grid(row=2, pady=10)

        Button(f2, text="Cancel", font=("courier", 16),
               command=lambda: self.home(f2)).grid(row=3, pady=10)

        f2.grid()

    def frame2_encode(self, f2):
        ep = Frame(self.root)
        video_path = tkinter.filedialog.askopenfilename(
            filetypes=[("Video Files", "*.mp4 *.avi *.mov *.mkv"), ("All Files", "*.*")]
        )
        if not video_path:
            messagebox.showerror("Error", "You have selected nothing!")
            return

        Label(ep, text="Selected Video", font=("courier", 16)).grid(row=0, pady=10)

        cap = cv2.VideoCapture(video_path)
        ok, frame = cap.read()
        cap.release()
        if ok:
            frame_rgb = cv2.cvtColor(frame, cv2.COLOR_BGR2RGB)
            img = Image.fromarray(frame_rgb).resize((360, 220))
            tkimg = ImageTk.PhotoImage(img)
            panel = Label(ep, image=tkimg)
            panel.image = tkimg
            panel.grid(row=1, pady=10)

        Label(ep, text="Enter the message", font=("courier", 16)).grid(row=2, pady=10)
        text_area = Text(ep, width=55, height=10)
        text_area.grid(row=3, pady=5)

        Button(ep, text="Encode", font=("courier", 12),
               command=lambda: self.encode_video_flow(text_area, video_path, ep)).grid(row=4, pady=12)

        Button(ep, text="Cancel", font=("courier", 12),
               command=lambda: self.home(ep)).grid(row=5, pady=8)

        ep.grid(row=1)
        f2.destroy()

    # ---------------- GUI: DECODE ----------------
    def frame1_decode(self, f):
        f.destroy()
        d_f2 = Frame(self.root)

        Label(d_f2, text="DECODE", font=("courier", 60)).grid(row=0, pady=30)
        Label(d_f2, text="Select Video with Hidden text:", font=("courier", 16)).grid(row=1, pady=10)

        Button(d_f2, text="Select", font=("courier", 16),
               command=lambda: self.frame2_decode(d_f2)).grid(row=2, pady=10)

        Button(d_f2, text="Cancel", font=("courier", 16),
               command=lambda: self.home(d_f2)).grid(row=3, pady=10)

        d_f2.grid()

    def frame2_decode(self, d_f2):
        d_f3 = Frame(self.root)
        video_path = tkinter.filedialog.askopenfilename(
            filetypes=[("AVI (recommended)", "*.avi"), ("Video Files", "*.mp4 *.avi *.mov *.mkv"), ("All Files", "*.*")]
        )
        if not video_path:
            messagebox.showerror("Error", "Nothing Selected")
            return

        Label(d_f3, text="Selected Video", font=("courier", 16)).grid(row=0, pady=10)

        cap = cv2.VideoCapture(video_path)
        ok, frame = cap.read()
        cap.release()
        if ok:
            frame_rgb = cv2.cvtColor(frame, cv2.COLOR_BGR2RGB)
            img = Image.fromarray(frame_rgb).resize((360, 220))
            tkimg = ImageTk.PhotoImage(img)
            panel = Label(d_f3, image=tkimg)
            panel.image = tkimg
            panel.grid(row=1, pady=10)

        hidden = self.decode_video(video_path)

        Label(d_f3, text="Hidden data is:", font=("courier", 16)).grid(row=2, pady=10)
        text_area = Text(d_f3, width=55, height=10)
        text_area.insert(INSERT, hidden)
        text_area.grid(row=3, pady=5)

        Button(d_f3, text="Cancel", font=("courier", 16),
               command=lambda: self.home(d_f3)).grid(row=4, pady=15)

        d_f3.grid(row=1)
        d_f2.destroy()

    # ---------------- STEG CORE ----------------
    def _bytes_to_bits(self, b: bytes) -> str:
        return "".join(f"{byte:08b}" for byte in b)

    def _bits_to_bytes(self, bits: str) -> bytes:
        out = bytearray()
        for i in range(0, len(bits), 8):
            chunk = bits[i:i+8]
            if len(chunk) < 8:
                break
            out.append(int(chunk, 2))
        return bytes(out)

    def _embed_bits_in_frame(self, frame_bgr, bits: str, bit_index: int):
        h, w, _ = frame_bgr.shape
        flat = frame_bgr.reshape(-1)
        capacity = flat.size
        remaining = len(bits) - bit_index
        take = min(remaining, capacity)
        if take <= 0:
            return frame_bgr, bit_index, True

        for i in range(take):
            flat[i] = (flat[i] & 0xFE) | int(bits[bit_index + i])

        bit_index += take
        done = (bit_index >= len(bits))
        return flat.reshape((h, w, 3)), bit_index, done

    def _extract_frame_lsb_bits(self, frame_bgr):
        flat = frame_bgr.reshape(-1)
        return "".join("1" if (v & 1) else "0" for v in flat)

    def encode_video_flow(self, text_area, video_path, frame):
        msg = text_area.get("1.0", "end-1c")
        if not msg:
            messagebox.showinfo("Alert", "Kindly enter text in TextBox")
            return

        # IMPORTANT: Use AVI + FFV1 (lossless) so decoding works.
        out_path = tkinter.filedialog.asksaveasfilename(
            initialfile=os.path.splitext(os.path.basename(video_path))[0] + "_steg.avi",
            defaultextension=".avi",
            filetypes=[("AVI (lossless recommended)", "*.avi"), ("All Files", "*.*")]
        )
        if not out_path:
            return

        try:
            self.encode_video(video_path, out_path, msg)
            messagebox.showinfo("Success", f"Encoding Successful!\nSaved as:\n{out_path}\n\nDecode THIS output file (not the original).")
            self.home(frame)
        except Exception as e:
            messagebox.showerror("Error", f"Encoding failed:\n{e}")

    def encode_video(self, in_path, out_path, message: str):
        payload = message.encode("utf-8")
        header = MAGIC + len(payload).to_bytes(4, "big")  # 8 bytes total
        data = header + payload
        bits = self._bytes_to_bits(data)

        cap = cv2.VideoCapture(in_path)
        if not cap.isOpened():
            raise RuntimeError("Could not open input video.")

        fps = cap.get(cv2.CAP_PROP_FPS) or 25.0
        width = int(cap.get(cv2.CAP_PROP_FRAME_WIDTH))
        height = int(cap.get(cv2.CAP_PROP_FRAME_HEIGHT))

        # Lossless codec (FFV1) in AVI container
        fourcc = cv2.VideoWriter_fourcc(*"FFV1")
        writer = cv2.VideoWriter(out_path, fourcc, fps, (width, height))
        if not writer.isOpened():
            cap.release()
            raise RuntimeError("Could not open output writer. Try a different output path or codec.")

        bit_index = 0
        done = False

        while True:
            ok, frame = cap.read()
            if not ok:
                break

            if not done:
                frame, bit_index, done = self._embed_bits_in_frame(frame, bits, bit_index)

            writer.write(frame)

        cap.release()
        writer.release()

        if not done:
            raise RuntimeError("Message too large for this video (not enough capacity).")

    def decode_video(self, in_path):
        cap = cv2.VideoCapture(in_path)
        if not cap.isOpened():
            return "Could not open video."

        bits_buffer = ""
        stage = "header"
        needed_bits = 8 * 8  # 8 bytes header = 64 bits
        payload_len = None

        try:
            while True:
                ok, frame = cap.read()
                if not ok:
                    break

                bits_buffer += self._extract_frame_lsb_bits(frame)

                while len(bits_buffer) >= needed_bits:
                    chunk = bits_buffer[:needed_bits]
                    bits_buffer = bits_buffer[needed_bits:]

                    if stage == "header":
                        header_bytes = self._bits_to_bytes(chunk)
                        if len(header_bytes) < 8:
                            return "No hidden message found."

                        if header_bytes[:4] != MAGIC:
                            return "No hidden message found (magic not present)."

                        payload_len = int.from_bytes(header_bytes[4:8], "big")
                        if payload_len <= 0 or payload_len > 5_000_000:
                            return "Hidden header found but payload length looks invalid."

                        stage = "payload"
                        needed_bits = payload_len * 8

                    elif stage == "payload":
                        payload_bytes = self._bits_to_bytes(chunk)
                        try:
                            return payload_bytes.decode("utf-8")
                        except:
                            return "Hidden data found but could not decode as UTF-8 (corrupted or wrong file)."

            return "No hidden message found."
        finally:
            cap.release()


if __name__ == "__main__":
    root = Tk()
    app = jalalvid()
    app.main(root)
    root.mainloop()



