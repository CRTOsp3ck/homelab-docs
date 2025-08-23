### **1. Using `dpkg` (low-level tool)**

```bash
sudo dpkg -i package-name.deb
```

* `-i` = install
* If dependencies are missing, fix them with:

```bash
sudo apt-get install -f
```

---

### **2. Using `apt` (preferred, handles dependencies better)**

Since modern `apt` supports installing `.deb` directly:

```bash
sudo apt install ./package-name.deb
```

* The `./` is needed to tell `apt` it’s a local file.
* `apt` automatically resolves dependencies.

---

### **3. Using `gdebi` (lightweight installer for .deb)**

Install `gdebi` first:

```bash
sudo apt-get install gdebi
```

Then:

```bash
sudo gdebi package-name.deb
```

* It’s dependency-aware and lighter than `apt`.

---

### **4. Using `dpkg-query` (to verify installation)**

After installation, check if the package is installed:

```bash
dpkg -l | grep package-name
```

---

✅ **Best practice**: Use `apt install ./package.deb` because it automatically resolves dependencies.
