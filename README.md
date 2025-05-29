# Refleksi Hello Minikube Saya

Setelah menyelesaikan tutorial "Hello Minikube", ini adalah beberapa hal yang saya pikirkan dan pelajari:

## 1. Perbandingan Log Aplikasi Sebelum dan Sesudah Saya Ekspos sebagai Service

Saat saya pertama kali men-deploy aplikasi (misalnya, pod web server), saya bisa melihat lognya menggunakan `kubectl logs <nama-pod-saya>`. Sebelum saya mengeksposnya sebagai Service dan mengaksesnya, log tersebut mungkin hanya berisi pesan startup aplikasi atau aktivitas internal yang tidak terkait dengan permintaan eksternal.

**Apa yang saya lihat di log setelah Service berjalan dan saya akses?**

Setelah saya mengekspos aplikasi sebagai Service (misalnya, dengan `kubectl expose deployment ...`) dan kemudian mengaksesnya, entah melalui `minikube service <nama-service-saya>` atau `kubectl port-forward svc/<nama-service-saya> ...`, saya mulai melihat hal berbeda di log Pod.

Setiap kali saya membuka aplikasi di browser atau melakukan refresh (yang pada dasarnya adalah permintaan HTTP GET baru), saya melihat entri log baru muncul di Pod yang menangani permintaan tersebut. Entri ini biasanya mencakup detail seperti:
*   Metode HTTP (misalnya, `GET`)
*   Path yang diminta (misalnya, `/`)
*   Alamat IP klien (mungkin alamat IP internal dari proxy atau dari dalam jaringan Minikube, tergantung cara saya mengaksesnya)
*   Kode status respons (misalnya, `200 OK`)
*   User-agent browser saya.

**Apakah jumlah log bertambah setiap kali saya membuka aplikasi?**

*   **Ya, tentu saja.** Setiap kali saya membuka aplikasi atau melakukan refresh halaman di browser, itu menghasilkan permintaan baru ke Service, yang kemudian meneruskannya ke salah satu Pod aplikasi saya. Pod tersebut kemudian mencatat permintaan masuk ini. Jadi, semakin sering saya mengaksesnya, semakin banyak baris log yang saya lihat. Ini sangat berguna untuk melihat bagaimana aplikasi saya merespons trafik.

## 2. Tujuan Opsi `-n` pada `kubectl get`

Saya perhatikan ada dua cara pemanggilan `kubectl get` dalam tutorial ini:

1.  `kubectl get pods` (atau `kubectl get services`, `kubectl get deployment`, dll.)
2.  `kubectl get pods -n kube-system`

**Apa tujuan dari opsi `-n`?**

Opsi `-n` (atau bentuk panjangnya `--namespace`) digunakan untuk **menentukan namespace** di mana `kubectl` harus mencari resource yang diminta. Namespace di Kubernetes itu seperti folder virtual di dalam cluster, yang memungkinkan saya (atau tim yang berbeda) untuk mengorganisir dan mengisolasi resource.

*   Ketika saya menjalankan `kubectl get pods` **tanpa** opsi `-n`, secara default `kubectl` akan mencari resource di namespace yang sedang aktif, yang biasanya adalah namespace `default`. Inilah mengapa saya bisa melihat pod `hello-minikube` (atau apa pun nama deployment yang saya buat) karena secara default, resource yang saya buat tanpa menentukan namespace akan masuk ke namespace `default`.

*   Ketika saya menjalankan `kubectl get pods -n kube-system`, saya secara eksplisit memberitahu `kubectl` untuk **hanya** menampilkan pod yang ada di dalam namespace `kube-system`.

**Mengapa output `kubectl get pods -n kube-system` tidak mencantumkan pod/service yang saya buat secara eksplisit?**

Output dari `kubectl get pods -n kube-system` tidak mencantumkan pod atau service yang saya buat (seperti pod dari deployment `hello-minikube`) karena:

1.  Pod dan service yang saya buat dalam tutorial ini (karena saya tidak menentukan namespace lain) ditempatkan di namespace `default`.
2.  Perintah `kubectl get pods -n kube-system` secara spesifik hanya mencari dan menampilkan resource yang ada di dalam namespace `kube-system`.
3.  Namespace `kube-system` adalah namespace khusus yang digunakan oleh Kubernetes itu sendiri untuk menjalankan komponen-komponen sistem intinya, seperti API server, etcd, DNS cluster (CoreDNS), controller manager, scheduler, dan agen kube-proxy di setiap node. Aplikasi pengguna seperti yang saya deploy biasanya tidak ditempatkan di sini.

Jadi, karena pod dan service aplikasi saya berada di namespace `default` dan perintah tersebut secara spesifik meminta resource dari `kube-system`, maka wajar jika resource saya tidak terdaftar dalam output tersebut. Ini membantu saya memahami bahwa Kubernetes menjaga komponen sistemnya terpisah dari aplikasi yang saya jalankan.

## Reflection on Rolling Update & Kubernetes Manifest File

Setelah menyelesaikan bagian tutorial ini, saya jadi lebih paham tentang bagaimana Kubernetes menangani pembaruan aplikasi dan betapa bergunanya manifest file. Berikut adalah refleksi saya:

### 1. Perbedaan Antara Strategi Deployment Rolling Update dan Recreate

Berdasarkan pengalaman saya dan dokumentasi Kubernetes, perbedaan utama antara strategi deployment **Rolling Update** dan **Recreate** adalah sebagai berikut:

*   **Rolling Update (Default Strategy):**
    *   **Proses:** Kubernetes secara bertahap mengganti instance Pod versi lama dengan instance Pod versi baru. Artinya, selama proses update, akan ada Pod versi lama dan versi baru yang berjalan secara bersamaan (jumlahnya tergantung konfigurasi `maxSurge` dan `maxUnavailable`).
    *   **Downtime:** Idealnya, tidak ada downtime (zero downtime) karena selalu ada Pod yang aktif untuk melayani permintaan selama transisi.
    *   **Resource:** Membutuhkan resource lebih sementara karena Pod lama dan baru berjalan bersamaan.
    *   **Rollback:** Lebih aman dan cepat untuk melakukan rollback jika ada masalah dengan versi baru, karena versi lama masih ada atau baru saja dihentikan.
    *   **Contoh Kasus:** Cocok untuk sebagian besar aplikasi produksi di mana ketersediaan tinggi adalah prioritas. Ini yang saya alami saat melakukan `kubectl set image`.

*   **Recreate Strategy:**
    *   **Proses:** Kubernetes akan mematikan *semua* instance Pod versi lama terlebih dahulu, baru kemudian membuat *semua* instance Pod versi baru.
    *   **Downtime:** Akan ada downtime aplikasi selama periode antara Pod lama dimatikan dan Pod baru siap sepenuhnya.
    *   **Resource:** Tidak membutuhkan resource tambahan sebanyak Rolling Update karena Pod lama dan baru tidak berjalan bersamaan.
    *   **Rollback:** Rollback tetap bisa dilakukan, tetapi karena aplikasi sudah mati total sebelum versi baru dimulai, dampaknya bisa lebih terasa jika versi baru bermasalah.
    *   **Contoh Kasus:** Mungkin cocok untuk aplikasi non-kritis, saat melakukan perubahan skema database yang tidak kompatibel antar versi, atau saat resource sangat terbatas dan tidak bisa mengakomodasi Pod lama dan baru berjalan bersamaan.

### 2. Mencoba Deploy Spring Petclinic REST Menggunakan Strategi Recreate

Untuk mencoba strategi Recreate, saya melakukan langkah-langkah berikut:

1.  **Memodifikasi Manifest Deployment:**
    Saya mengambil file `deployment.yaml` yang sebelumnya saya ekspor untuk aplikasi `spring-petclinic-rest`. Kemudian, saya menambahkan/mengubah bagian `spec.strategy` menjadi seperti ini:
    ```yaml
    spec:
      # ... (bagian lain seperti replicas, selector)
      strategy:
        type: Recreate # Mengubah strategi menjadi Recreate
      template:
        # ... (definisi template Pod)
    ```
    Saya menyimpan perubahan ini.

2.  **Menghapus Deployment yang Ada (jika perlu):**
    Untuk memastikan saya melihat efek Recreate dengan jelas, saya menghapus deployment `spring-petclinic-rest` yang mungkin sudah berjalan dengan strategi Rolling Update:
    ```bash
    kubectl delete deployment spring-petclinic-rest
    # Tunggu sampai semua pod terkait terhapus
    kubectl get pods
    ```

3.  **Menerapkan Manifest dengan Strategi Recreate:**
    Setelah itu, saya menerapkan manifest yang sudah dimodifikasi:
    ```bash
    kubectl apply -f deployment-recreate.yaml
    ```

4.  **Mengamati Proses Rollout:**
    Saya memantau status pod menggunakan:
    ```bash
    kubectl get pods -w
    ```
    Jika sebelumnya ada pod versi lama, saya akan melihat semua pod lama dihentikan (`Terminating`) terlebih dahulu. Setelah itu, barulah pod-pod baru dengan versi image yang sama atau berbeda (jika saya juga mengubah image) mulai dibuat (`Pending` -> `ContainerCreating` -> `Running`). Ada jeda waktu di mana tidak ada pod `spring-petclinic-rest` yang `Running`.

5.  **Dokumentasi Upaya:**
    ![Image](https://github.com/user-attachments/assets/fffd9fad-cb74-4cee-9dc5-a234c5b84fcd)


### 3. Menyiapkan Manifest File Berbeda untuk Strategi Recreate

Saya telah membuat file baru bernama `deployment-recreate.yaml` sebagai salinan dari `deployment.yaml` yang menggunakan strategi Recreate.

### 4. Manfaat Menggunakan Kubernetes Manifest Files
Setelah mencoba deploy manual dengan serangkaian perintah kubectl create/expose/scale/set image dan kemudian menggunakan `kubectl apply -f` dengan manifest file, saya merasakan beberapa manfaat dari penggunaan manifest file:
- Declarative Approach (Pendekatan Deklaratif):
Dengan manifest file, saya mendefinisikan keadaan yang diinginkan (desired state) dari aplikasi saya, bukan serangkaian perintah imperatif. Kubernetes kemudian yang bertugas untuk mencapai keadaan tersebut. Ini lebih mudah dipahami dan dikelola.
- Version Control (Kontrol Versi):
Manifest file (YAML) adalah teks biasa, jadi sangat mudah untuk disimpan dalam sistem kontrol versi seperti Git. Saya bisa melacak perubahan, melihat histori, berkolaborasi dengan tim, dan kembali ke versi konfigurasi sebelumnya jika diperlukan. Ini sangat sulit dilakukan dengan perintah kubectl manual.
- Repeatability & Consistency (Keterulangan & Konsistensi):
Saya bisa menggunakan manifest file yang sama untuk mendeploy aplikasi di lingkungan yang berbeda (development, staging, production) atau untuk membuat ulang deployment dengan cepat jika cluster direset. Ini memastikan konsistensi dan mengurangi risiko kesalahan manual.
- Idempotency (Idempotensi):
Perintah kubectl apply -f bersifat idempoten. Artinya, saya bisa menjalankannya berkali-kali dengan manifest yang sama, dan Kubernetes hanya akan melakukan perubahan jika keadaan saat ini berbeda dari yang didefinisikan di file. Ini aman dan praktis.
Readability & Documentation (Keterbacaan & Dokumentasi):
Manifest file berfungsi sebagai dokumentasi hidup dari konfigurasi deployment saya. Siapa pun bisa membacanya untuk memahami bagaimana aplikasi di-deploy dan dikonfigurasi.
- Automation (Otomatisasi):
Manifest file adalah dasar untuk otomatisasi CI/CD. Pipeline bisa secara otomatis menerapkan manifest file untuk mendeploy atau memperbarui aplikasi.
Complexity Management (Manajemen Kompleksitas):
Untuk aplikasi yang kompleks dengan banyak resource (Deployments, Services, ConfigMaps, Secrets, dll.), mengelola semuanya dengan perintah manual akan sangat merepotkan dan rentan error. Manifest file (atau kumpulan manifest) membuat ini jauh lebih terstruktur.

---