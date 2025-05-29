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

---
