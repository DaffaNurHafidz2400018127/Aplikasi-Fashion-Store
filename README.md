# Aplikasi-Fashion-Store
Tugas Project Sdata Kelompok 7
#include <stdio.h>
#include <stdlib.h>
#include <stdbool.h>
#include <time.h>
#define MAX_PRODUCTS 100
#define MAX_USERS 50
#define MAX_CART 20
#define MAX_FAVORITES 30
#define MAX_NAME 100
#define MAX_DESC 200

// Tambahan untuk Stack & Queue
#define MAX_VIEWED_STACK 50
#define MAX_ORDERS 50

// Struktur Data Produk
typedef struct {
    int id;
    char name[MAX_NAME];
    char category[50]; // women, man, kids, baby, electronics, ...
    char size[20];
    char color[30];
    double price;
    int stockOnline;
    int stockOffline;
    double rating;
    int ratingCount;
} Product;

// Struktur Data User
typedef struct {
    int id;
    char username[MAX_NAME];
    char password[MAX_NAME];
    char role[20]; // superadmin, admin, pembeli
    bool isLoggedIn;
} User;

// Struktur Data Keranjang
typedef struct {
    int productId;
    int quantity;
    char storeType[20]; // online/offline
} CartItem;

// Struktur Data Favorit
typedef struct {
    int productId;
} Favorite;

// Struktur Order untuk Queue
typedef struct {
    int orderId;
    int buyerUserId; // index di users[]
    double total;
    int itemCount;
    // simple: simpan productId dan qty sebagai arrays kecil
    int productIds[MAX_CART];
    int quantities[MAX_CART];
    time_t timestamp;
} Order;

// Linked list node untuk riwayat transaksi (simple)
typedef struct TransactionNode {
    int buyerUserId;
    double total;
    int itemCount;
    int productIds[MAX_CART];
    int quantities[MAX_CART];
    time_t timestamp;
    struct TransactionNode *next;
} TransactionNode;

// Queue untuk pengembalian barang (return requests)
typedef struct {
    int requestId;
    int userId;
    int productId;
    int quantity;
    time_t timestamp;
} ReturnRequest;

// Graph adjacency for product category relations (simple)
#define MAX_CATEGORIES 10
typedef struct {
    char name[50];
} CategoryNode;

typedef struct {
    CategoryNode nodes[MAX_CATEGORIES];
    int adj[MAX_CATEGORIES][MAX_CATEGORIES];
    int count;
} CategoryGraph;

// Global Variables
Product products[MAX_PRODUCTS];
int productCount = 0;

User users[MAX_USERS];
int userCount = 0;
int currentUserId = -1;

CartItem cart[MAX_CART];
int cartCount = 0;

Favorite favorites[MAX_FAVORITES];
int favoriteCount = 0;

// Stack: recently viewed products (store product IDs)
int viewedStack[MAX_VIEWED_STACK];
int viewedTop = -1;

// Queue: orders
Order orderQueue[MAX_ORDERS];
int orderFront = 0;
int orderRear = 0;
int orderCount = 0;
int nextOrderId = 1; // increment setiap dibuat order

// Transaction linked list head (riwayat)
TransactionNode *transactionHead = NULL;

// Return queue
ReturnRequest returnQueue[MAX_ORDERS];
int returnFront = 0;
int returnRear = 0;
int returnCount = 0;
int nextReturnId = 1;

// Login history stack (store userId)
int loginHistory[10];
int loginHistoryTop = -1;

// Category graph
CategoryGraph catGraph;

// Fungsi Helper
void clearScreen() {
    #ifdef _WIN32
        system("cls");
    #else
        system("clear");
    #endif
}

void pause() {
    printf("\nTekan Enter untuk melanjutkan...");
    getchar();
    // consume leftover newline if exists
    // second getchar to ensure pause works if previous input used scanf
    // but avoid blocking twice when not needed
    // Safe approach: try one more getchar
    // Note: On some systems a single getchar() is enough
    getchar();
}

void strcpy_safe(char *dest, const char *src, int maxLen) {
    int i = 0;
    while (src[i] != '\0' && i < maxLen - 1) {
        dest[i] = src[i];
        i++;
    }
    dest[i] = '\0';
}

int strcmp_custom(const char *str1, const char *str2) {
    int i = 0;
    while (str1[i] != '\0' && str2[i] != '\0') {
        if (str1[i] != str2[i]) {
            return str1[i] - str2[i];
        }
        i++;
    }
    return str1[i] - str2[i];
}

bool strcontains(const char *haystack, const char *needle) {
    int i = 0, j = 0;
    if (needle[0] == '\0') return true;
    while (haystack[i] != '\0') {
        if (haystack[i] == needle[j]) {
            j++;
            if (needle[j] == '\0') return true;
        } else {
            j = 0;
        }
        i++;
    }
    return false;
}

void toLowerCase(char *str) {
    for (int i = 0; str[i] != '\0'; i++) {
        if (str[i] >= 'A' && str[i] <= 'Z') {
            str[i] = str[i] + 32;
        }
    }
}

// ---------------- Category Graph Functions ----------------
void initCategoryGraph() {
    catGraph.count = 0;
}

int findCategoryIndex(const char *name) {
    for (int i = 0; i < catGraph.count; i++) {
        if (strcmp_custom(catGraph.nodes[i].name, name) == 0) return i;
    }
    return -1;
}

void addCategoryIfNotExists(const char *name) {
    if (findCategoryIndex(name) != -1) return;
    if (catGraph.count < MAX_CATEGORIES) {
        strcpy_safe(catGraph.nodes[catGraph.count].name, name, 50);
        for (int i = 0; i < MAX_CATEGORIES; i++) catGraph.adj[catGraph.count][i] = 0;
        for (int i = 0; i < MAX_CATEGORIES; i++) catGraph.adj[i][catGraph.count] = 0;
        catGraph.count++;
    }
}

void addCategoryEdge(const char *a, const char *b) {
    addCategoryIfNotExists(a);
    addCategoryIfNotExists(b);
    int ia = findCategoryIndex(a);
    int ib = findCategoryIndex(b);
    if (ia != -1 && ib != -1) {
        catGraph.adj[ia][ib] = 1;
        catGraph.adj[ib][ia] = 1;
    }
}

void showCategoryGraph() {
    clearScreen();
    printf("=== GRAF HUBUNGAN KATEGORI PRODUK ===\n\n");
    for (int i = 0; i < catGraph.count; i++) {
        printf("%s ->", catGraph.nodes[i].name);
        bool first = true;
        for (int j = 0; j < catGraph.count; j++) {
            if (catGraph.adj[i][j]) {
                if (!first) printf(", ");
                printf(" %s", catGraph.nodes[j].name);
                first = false;
            }
        }
        printf("\n");
    }
    pause();
}

// ---------------- Transaction Linked List ----------------
void addTransactionToList(Order *o) {
    TransactionNode *node = (TransactionNode*) malloc(sizeof(TransactionNode));
    if (!node) return;
    node->buyerUserId = o->buyerUserId;
    node->total = o->total;
    node->itemCount = o->itemCount;
    for (int i = 0; i < o->itemCount; i++) {
        node->productIds[i] = o->productIds[i];
        node->quantities[i] = o->quantities[i];
    }
    node->timestamp = o->timestamp;
    node->next = transactionHead;
    transactionHead = node;
}

void showTransactionsByUser(int userIndex) {
    clearScreen();
    printf("=== RIWAYAT TRANSAKSI USER (ID index: %d) ===\n\n", userIndex);
    TransactionNode *curr = transactionHead;
    if (!curr) {
        printf("Belum ada transaksi.\n");
        pause();
        return;
    }
    int count = 0;
    while (curr) {
        if (curr->buyerUserId == userIndex) {
            char tbuf[26];
            struct tm *tm_info = localtime(&curr->timestamp);
            strftime(tbuf, 26, "%Y-%m-%d %H:%M:%S", tm_info);
            printf("Transaksi #%d | Total: Rp %.0f | Items: %d | Time: %s\n",
                   ++count, curr->total, curr->itemCount, tbuf);
            for (int i = 0; i < curr->itemCount; i++) {
                int pid = curr->productIds[i];
                int qty = curr->quantities[i];
                for (int p = 0; p < productCount; p++) {
                    if (products[p].id == pid) {
                        printf("   - %s (ID:%d) x%d\n", products[p].name, pid, qty);
                        break;
                    }
                }
            }
            printf("---------------------------------\n");
        }
        curr = curr->next;
    }
    if (count == 0) printf("Belum ada transaksi untuk user ini.\n");
    pause();
}

// ---------------- Return Queue ----------------
bool enqueueReturn(ReturnRequest r) {
    if (returnCount >= MAX_ORDERS) return false;
    returnQueue[returnRear] = r;
    returnRear = (returnRear + 1) % MAX_ORDERS;
    returnCount++;
    return true;
}

bool dequeueReturn(ReturnRequest *out) {
    if (returnCount == 0) return false;
    *out = returnQueue[returnFront];
    returnFront = (returnFront + 1) % MAX_ORDERS;
    returnCount--;
    return true;
}

void viewReturnQueue() {
    clearScreen();
    printf("=== QUEUE PENGEMBALIAN BARANG ===\n\n");
    if (returnCount == 0) {
        printf("Belum ada permintaan pengembalian.\n");
        pause();
        return;
    }
    int idx = returnFront;
    for (int i = 0; i < returnCount; i++) {
        ReturnRequest *r = &returnQueue[idx];
        char tbuf[26];
        struct tm *tm_info = localtime(&r->timestamp);
        strftime(tbuf, 26, "%Y-%m-%d %H:%M:%S", tm_info);
        printf("RequestID: %d | UserIdx: %d | ProductID: %d | Qty: %d | Time: %s\n",
               r->requestId, r->userId, r->productId, r->quantity, tbuf);
        idx = (idx + 1) % MAX_ORDERS;
    }
    printf("\n1. Proses next return (dequeue)\n0. Kembali\nPilihan: ");
    int ch;
    scanf("%d", &ch);
    if (ch == 1) {
        ReturnRequest r;
        if (dequeueReturn(&r)) {
            printf("Return Request %d diproses (dequeued).\n", r.requestId);
            // optionally restore stock - not implemented automatically
        } else {
            printf("Tidak ada request untuk diproses.\n");
        }
        pause();
    }
}

// ---------------- Login History Stack ----------------
void pushLoginHistory(int userIdx) {
    if (loginHistoryTop >= 9) {
        // shift to make room (keep last 10)
        for (int i = 0; i < 9; i++) loginHistory[i] = loginHistory[i+1];
        loginHistoryTop = 8;
    }
    loginHistoryTop++;
    loginHistory[loginHistoryTop] = userIdx;
}

int popLoginHistory() {
    if (loginHistoryTop < 0) return -1;
    int val = loginHistory[loginHistoryTop];
    loginHistoryTop--;
    return val;
}

void viewLoginHistory() {
    clearScreen();
    printf("=== RIWAYAT LOGIN (TERAKHIR) ===\n\n");
    if (loginHistoryTop < 0) {
        printf("Belum ada riwayat login.\n");
        pause();
        return;
    }
    for (int i = loginHistoryTop; i >= 0; i--) {
        int idx = loginHistory[i];
        printf("%d. UserIndex: %d | Username: %s\n", loginHistoryTop - i + 1, idx, users[idx].username);
    }
    pause();
}

// Inisialisasi Data Default
void initializeData() {
    initCategoryGraph();

    // User Default
    strcpy_safe(users[0].username, "superadmin", MAX_NAME);
    strcpy_safe(users[0].password, "super123", MAX_NAME);
    strcpy_safe(users[0].role, "superadmin", 20);
    users[0].id = 1;
    users[0].isLoggedIn = false;

    strcpy_safe(users[1].username, "admin", MAX_NAME);
    strcpy_safe(users[1].password, "admin123", MAX_NAME);
    strcpy_safe(users[1].role, "admin", 20);
    users[1].id = 2;
    users[1].isLoggedIn = false;

    strcpy_safe(users[2].username, "pembeli", MAX_NAME);
    strcpy_safe(users[2].password, "beli123", MAX_NAME);
    strcpy_safe(users[2].role, "pembeli", 20);
    users[2].id = 3;
    users[2].isLoggedIn = false;

    userCount = 3;

    // Produk Women
    products[0] = (Product){1, "Dress Elegant", "women", "M", "Black", 250000, 15, 10, 4.5, 20};
    products[1] = (Product){2, "Blouse Casual", "women", "L", "White", 150000, 20, 15, 4.2, 15};
    products[2] = (Product){3, "Skirt Mini", "women", "S", "Red", 120000, 10, 8, 4.0, 12};

    // Produk Man
    products[3] = (Product){4, "Kemeja Formal", "man", "L", "Blue", 200000, 18, 12, 4.3, 18};
    products[4] = (Product){5, "Celana Jeans", "man", "32", "Black", 300000, 25, 20, 4.7, 30};
    products[5] = (Product){6, "Kaos Polo", "man", "M", "Gray", 100000, 30, 25, 4.1, 25};

    // Produk Kids
    products[6] = (Product){7, "T-Shirt Anak", "kids", "8-10", "Yellow", 80000, 20, 15, 4.4, 10};
    products[7] = (Product){8, "Celana Pendek", "kids", "6-8", "Blue", 90000, 15, 10, 4.2, 8};
    products[8] = (Product){9, "Dress Anak", "kids", "10-12", "Pink", 130000, 12, 8, 4.6, 15};

    // Produk Baby
    products[9] = (Product){10, "Romper Baby", "baby", "0-6m", "White", 120000, 10, 8, 4.8, 20};
    products[10] = (Product){11, "Baju Tidur Baby", "baby", "6-12m", "Blue", 100000, 15, 10, 4.5, 12};
    products[11] = (Product){12, "Jumper Baby", "baby", "12-18m", "Green", 110000, 12, 9, 4.3, 10};

    // Tambah Produk Elektronik (kategori electronics)
//    products[12] = (Product){13, "Laptop ASUS A416", "electronics", "14 inch", "Silver", 8500000, 5, 2, 4.6, 50};
//    products[13] = (Product){14, "Smartphone XPlus", "electronics", "6.5 inch", "Black", 4500000, 10, 5, 4.4, 40};
//    products[14] = (Product){15, "Headset Bluetooth", "electronics", "-", "Black", 350000, 25, 15, 4.2, 30};
//    products[15] = (Product){16, "Blender Dapur", "electronics", "-", "White", 500000, 8, 4, 4.0, 12};

    productCount = 16;

    // graph categories relations (example)
    addCategoryIfNotExists("women");
    addCategoryIfNotExists("man");
    addCategoryIfNotExists("kids");
    addCategoryIfNotExists("baby");
//    addCategoryIfNotExists("electronics");
//    addCategoryIfNotExists("accessories");
//    addCategoryEdge("electronics", "accessories");
    addCategoryEdge("women", "accessories");
    addCategoryEdge("man", "accessories");

    // init stack & queue vars
    viewedTop = -1;
    orderFront = 0;
    orderRear = 0;
    orderCount = 0;
    nextOrderId = 1;

    // return queue init
    returnFront = 0;
    returnRear = 0;
    returnCount = 0;
    nextReturnId = 1;

    // login history init
    loginHistoryTop = -1;

    // transaction list init
    transactionHead = NULL;
}

// ---------------------- Stack (Recently Viewed) ----------------------
bool pushViewed(int productId) {
    if (viewedTop >= MAX_VIEWED_STACK - 1) return false; // penuh
    viewedTop++;
    viewedStack[viewedTop] = productId;
    return true;
}

int popViewed() {
    if (viewedTop < 0) return -1; // kosong
    int pid = viewedStack[viewedTop];
    viewedTop--;
    return pid;
}

void showViewedStack() {
    clearScreen();
    printf("=== Recently Viewed (STACK) ===\n\n");
    if (viewedTop < 0) {
        printf("Belum ada produk yang dilihat.\n");
        pause();
        return;
    }
    for (int i = viewedTop; i >= 0; i--) {
        int pid = viewedStack[i];
        // tampilkan nama produk
        for (int j = 0; j < productCount; j++) {
            if (products[j].id == pid) {
                printf("%d. %s (ID: %d)\n", viewedTop - i + 1, products[j].name, pid);
                break;
            }
        }
    }
    printf("\n1. Pop terakhir (hapus)\n2. Clear semua\n0. Kembali\nPilihan: ");
    int ch;
    scanf("%d", &ch);
    if (ch == 1) {
        int popped = popViewed();
        if (popped == -1) printf("Stack kosong.\n");
        else printf("Produk ID %d dihapus dari stack.\n", popped);
        pause();
    } else if (ch == 2) {
        viewedTop = -1;
        printf("Stack direset.\n");
        pause();
    }
}

// ---------------------- Queue (Order Queue) ----------------------
bool enqueueOrder(Order o) {
    if (orderCount >= MAX_ORDERS) return false;
    orderQueue[orderRear] = o;
    orderRear = (orderRear + 1) % MAX_ORDERS;
    orderCount++;
    return true;
}

bool dequeueOrder(Order *out) {
    if (orderCount == 0) return false;
    *out = orderQueue[orderFront];
    orderFront = (orderFront + 1) % MAX_ORDERS;
    orderCount--;
    return true;
}

void viewOrderQueue() {
    clearScreen();
    printf("=== ORDER QUEUE (ANTREAN PESANAN) ===\n\n");
    if (orderCount == 0) {
        printf("Belum ada pesanan di antrean.\n");
        pause();
        return;
    }
    int idx = orderFront;
    for (int i = 0; i < orderCount; i++) {
        Order *o = &orderQueue[idx];
        char tbuf[26];
        struct tm *tm_info = localtime(&o->timestamp);
        strftime(tbuf, 26, "%Y-%m-%d %H:%M:%S", tm_info);
        printf("OrderID: %d | Buyer user index: %d | Total: Rp %.0f | Items: %d | Time: %s\n",
               o->orderId, o->buyerUserId, o->total, o->itemCount, tbuf);
        for (int k = 0; k < o->itemCount; k++) {
            int pid = o->productIds[k];
            int qty = o->quantities[k];
            // find product name
            for (int p = 0; p < productCount; p++) {
                if (products[p].id == pid) {
                    printf("   - %s (ID:%d) x%d\n", products[p].name, pid, qty);
                    break;
                }
            }
        }
        printf("---------------------------------\n");
        idx = (idx + 1) % MAX_ORDERS;
    }
    printf("\n1. Proses next order (dequeue)\n0. Kembali\nPilihan: ");
    int ch;
    scanf("%d", &ch);
    if (ch == 1) {
        Order o;
        if (dequeueOrder(&o)) {
            printf("Order %d diproses (dequeued).\n", o.orderId);
        } else {
            printf("Tidak ada order untuk diproses.\n");
        }
        pause();
    }
}

// Fungsi Login & Authentication
void login() {
    clearScreen();
    printf("=== LOGIN FASHION STORE ===\n\n");
    
    char username[MAX_NAME], password[MAX_NAME];
    printf("Username: ");
    scanf("%s", username);
    printf("Password: ");
    scanf("%s", password);

    for (int i = 0; i < userCount; i++) {
        if (strcmp_custom(users[i].username, username) == 0 && 
            strcmp_custom(users[i].password, password) == 0) {
            users[i].isLoggedIn = true;
            currentUserId = i;
            pushLoginHistory(i);
            printf("\nLogin berhasil! Selamat datang %s (%s)\n", users[i].username, users[i].role);
            pause();
            return;
        }
    }

    printf("\nLogin gagal! Username atau password salah.\n");
    pause();
}

void logout() {
    if (currentUserId >= 0) {
        users[currentUserId].isLoggedIn = false;
        currentUserId = -1;
        cartCount = 0;
        favoriteCount = 0;
    }
    printf("\nLogout berhasil!\n");
    pause();
}

// Fungsi Menampilkan Produk
void displayProduct(Product p) {
    printf("ID: %d\n", p.id);
    printf("Nama: %s\n", p.name);
    printf("Kategori: %s\n", p.category);
    printf("Ukuran: %s | Warna: %s\n", p.size, p.color);
    printf("Harga: Rp %.0f\n", p.price);
    printf("Stok Online: %d | Offline: %d\n", p.stockOnline, p.stockOffline);
    printf("Rating: %.1f/5.0 (%d reviews)\n", p.rating, p.ratingCount);
    printf("----------------------------------------\n");

    // Tambahkan ke stack recently viewed kalau ada user yang login dan role pembeli
    if (currentUserId >= 0 && strcmp_custom(users[currentUserId].role, "pembeli") == 0) {
        pushViewed(p.id);
    }
}

void displayProductsByCategory(const char *category) {
    clearScreen();
    printf("=== PRODUK KATEGORI: %s ===\n\n", category);
    
    bool found = false;
    for (int i = 0; i < productCount; i++) {
        if (strcmp_custom(products[i].category, category) == 0) {
            displayProduct(products[i]);
            found = true;
        }
    }

    if (!found) {
        printf("Tidak ada produk dalam kategori ini.\n");
    }
    pause();
}

// ---------------- Sorting Produk ----------------
// helper: swap indices
void swap_int(int *a, int *b) {
    int tmp = *a; *a = *b; *b = tmp;
}

// show sorted (non-destructive) by comparator on products[indexes[i]]
// comparator: 1 = name asc, 2 = name desc, 3 = price asc, 4 = price desc
void showSortedProducts(int comparator) {
    if (productCount == 0) {
        printf("Belum ada produk.\n");
        pause();
        return;
    }
    int idxs[MAX_PRODUCTS];
    for (int i = 0; i < productCount; i++) idxs[i] = i;

    // simple bubble sort on idxs
    for (int i = 0; i < productCount - 1; i++) {
        for (int j = 0; j < productCount - i - 1; j++) {
            bool cond = false;
            if (comparator == 1) { // name asc
                if (strcmp_custom(products[idxs[j]].name, products[idxs[j+1]].name) > 0) cond = true;
            } else if (comparator == 2) { // name desc
                if (strcmp_custom(products[idxs[j]].name, products[idxs[j+1]].name) < 0) cond = true;
            } else if (comparator == 3) { // price asc
                if (products[idxs[j]].price > products[idxs[j+1]].price) cond = true;
            } else if (comparator == 4) { // price desc
                if (products[idxs[j]].price < products[idxs[j+1]].price) cond = true;
            }
            if (cond) swap_int(&idxs[j], &idxs[j+1]);
        }
    }

    clearScreen();
    printf("=== HASIL SORTING PRODUK ===\n\n");
    for (int i = 0; i < productCount; i++) {
        displayProduct(products[idxs[i]]);
    }
    pause();
}

void sortMenu() {
    clearScreen();
    printf("=== MENU SORTING PRODUK ===\n\n");
    printf("1. Nama A-Z\n");
    printf("2. Nama Z-A\n");
    printf("3. Harga Terendah ke Tertinggi\n");
    printf("4. Harga Tertinggi ke Terendah\n");
    printf("0. Kembali\n");
    printf("\nPilihan: ");
    int ch;
    scanf("%d", &ch);
    switch (ch) {
        case 1: showSortedProducts(1); break;
        case 2: showSortedProducts(2); break;
        case 3: showSortedProducts(3); break;
        case 4: showSortedProducts(4); break;
        default: break;
    }
}

// ---------------- Searching ----------------
void searchProduct() {
    clearScreen();
    printf("=== CARI PRODUK ===\n\n");
    
    char keyword[MAX_NAME];
    printf("Masukkan kata kunci: ");
    scanf(" %[^\n]", keyword);
    toLowerCase(keyword);

    printf("\n=== HASIL PENCARIAN ===\n\n");
    bool found = false;
    
    for (int i = 0; i < productCount; i++) {
        char tempName[MAX_NAME], tempCategory[50];
        strcpy_safe(tempName, products[i].name, MAX_NAME);
        strcpy_safe(tempCategory, products[i].category, 50);
        toLowerCase(tempName);
        toLowerCase(tempCategory);

        if (strcontains(tempName, keyword) || strcontains(tempCategory, keyword)) {
            displayProduct(products[i]);
            found = true;
        }
    }

    if (!found) {
        printf("Produk tidak ditemukan.\n");
    }
    pause();
}

// ---------------- Favorit ----------------
void addToFavorites() {
    clearScreen();
    printf("=== TAMBAH KE FAVORIT ===\n\n");
    
    int id;
    printf("Masukkan ID Produk: ");
    scanf("%d", &id);

    // Cek apakah produk ada
    bool productExists = false;
    for (int i = 0; i < productCount; i++) {
        if (products[i].id == id) {
            productExists = true;
            break;
        }
    }

    if (!productExists) {
        printf("Produk tidak ditemukan!\n");
        pause();
        return;
    }

    // Cek apakah sudah ada di favorit
    for (int i = 0; i < favoriteCount; i++) {
        if (favorites[i].productId == id) {
            printf("Produk sudah ada di favorit!\n");
            pause();
            return;
        }
    }

    if (favoriteCount < MAX_FAVORITES) {
        favorites[favoriteCount].productId = id;
        favoriteCount++;
        printf("Produk berhasil ditambahkan ke favorit!\n");
    } else {
        printf("Favorit penuh!\n");
    }
    pause();
}

void viewFavorites() {
    clearScreen();
    printf("=== DAFTAR FAVORIT ===\n\n");

    if (favoriteCount == 0) {
        printf("Favorit masih kosong.\n");
        pause();
        return;
    }

    for (int i = 0; i < favoriteCount; i++) {
        for (int j = 0; j < productCount; j++) {
            if (products[j].id == favorites[i].productId) {
                displayProduct(products[j]);
                break;
            }
        }
    }
    pause();
}

// ---------------- Keranjang & Checkout (dengan konfirmasi) ----------------
void addToCart() {
    clearScreen();
    printf("=== TAMBAH KE KERANJANG ===\n\n");

    int id, qty;
    char storeType[20];

    printf("Masukkan ID Produk: ");
    scanf("%d", &id);
    printf("Jumlah: ");
    scanf("%d", &qty);
    printf("Tipe Toko (online/offline): ");
    scanf("%s", storeType);

    // Cek produk dan stok
    bool found = false;
    for (int i = 0; i < productCount; i++) {
        if (products[i].id == id) {
            found = true;
            int availableStock = (strcmp_custom(storeType, "online") == 0) ? 
                                 products[i].stockOnline : products[i].stockOffline;
            
            if (qty > availableStock) {
                printf("Stok tidak cukup! Stok tersedia: %d\n", availableStock);
                pause();
                return;
            }
            break;
        }
    }

    if (!found) {
        printf("Produk tidak ditemukan!\n");
        pause();
        return;
    }

    if (cartCount < MAX_CART) {
        cart[cartCount].productId = id;
        cart[cartCount].quantity = qty;
        strcpy_safe(cart[cartCount].storeType, storeType, 20);
        cartCount++;
        printf("Produk berhasil ditambahkan ke keranjang!\n");
    } else {
        printf("Keranjang penuh!\n");
    }
    pause();
}

void viewCart() {
    clearScreen();
    printf("=== ISI KERANJANG ===\n\n");

    if (cartCount == 0) {
        printf("Keranjang masih kosong.\n");
        pause();
        return;
    }

    double total = 0;
    for (int i = 0; i < cartCount; i++) {
        for (int j = 0; j < productCount; j++) {
            if (products[j].id == cart[i].productId) {
                printf("%d. %s\n", i+1, products[j].name);
                printf("   Harga: Rp %.0f x %d = Rp %.0f\n", 
                       products[j].price, cart[i].quantity, 
                       products[j].price * cart[i].quantity);
                printf("   Toko: %s\n", cart[i].storeType);
                printf("----------------------------------------\n");
                total += products[j].price * cart[i].quantity;
                break;
            }
        }
    }

    printf("\nTotal: Rp %.0f\n", total);
    pause();
}

void checkout() {
    clearScreen();
    printf("=== CHECKOUT ===\n\n");

    if (cartCount == 0) {
        printf("Keranjang masih kosong!\n");
        pause();
        return;
    }

    double total = 0;
    printf("Ringkasan Pesanan:\n\n");

    // Mempersiapkan order untuk queue
    Order newOrder;
    newOrder.orderId = nextOrderId++;
    newOrder.buyerUserId = currentUserId;
    newOrder.itemCount = 0;
    newOrder.timestamp = time(NULL);

    for (int i = 0; i < cartCount; i++) {
        for (int j = 0; j < productCount; j++) {
            if (products[j].id == cart[i].productId) {
                printf("%s x%d - Rp %.0f\n", products[j].name, cart[i].quantity,
                       products[j].price * cart[i].quantity);
                total += products[j].price * cart[i].quantity;
                
                // Kurangi stok AFTER confirmation
                // Simpan detail order dulu
                newOrder.productIds[newOrder.itemCount] = products[j].id;
                newOrder.quantities[newOrder.itemCount] = cart[i].quantity;
                newOrder.itemCount++;

                break;
            }
        }
    }

    newOrder.total = total;

    printf("\nTotal Pembayaran: Rp %.0f\n", total);
    printf("\nPilih metode pembayaran:\n1. Transfer\n2. Kartu Kredit\n3. COD\nPilihan: ");
    int method;
    scanf("%d", &method);
    const char *methodName = (method == 1 ? "Transfer" : method == 2 ? "Kartu Kredit" : "COD");
    printf("\nMetode: %s\n", methodName);
    printf("Konfirmasi pembayaran? (y/n): ");
    char conf;
    scanf(" %c", &conf);
    if (conf == 'y' || conf == 'Y') {
        // kurangi stok sekarang
        for (int i = 0; i < newOrder.itemCount; i++) {
            int pid = newOrder.productIds[i];
            int qty = newOrder.quantities[i];
            for (int j = 0; j < productCount; j++) {
                if (products[j].id == pid) {
                    // reduce from online stock by default
                    if (products[j].stockOnline >= qty) products[j].stockOnline -= qty;
                    else products[j].stockOffline = (products[j].stockOffline >= qty ? products[j].stockOffline - qty : 0);
                    break;
                }
            }
        }

        printf("\nPembayaran berhasil! Terima kasih telah berbelanja.\n");
        // enqueue order
        if (!enqueueOrder(newOrder)) {
            printf("Peringatan: Order berhasil dibuat namun antrean penuh, tidak tersimpan di antrean.\n");
        } else {
            printf("Order dimasukkan ke antrean pemrosesan (OrderID: %d).\n", newOrder.orderId);
        }
        // simpan juga ke linked list transaksi
        addTransactionToList(&newOrder);
        cartCount = 0;
    } else {
        printf("\nPembayaran dibatalkan. Keranjang tetap tersimpan.\n");
    }
    pause();
}

// ---------------- Rating ----------------
void rateProduct() {
    clearScreen();
    printf("=== BERI PENILAIAN ===\n\n");

    int id;
    double rating;

    printf("Masukkan ID Produk: ");
    scanf("%d", &id);
    printf("Rating (1-5): ");
    scanf("%lf", &rating);

    if (rating < 1 || rating > 5) {
        printf("Rating harus antara 1-5!\n");
        pause();
        return;
    }

    for (int i = 0; i < productCount; i++) {
        if (products[i].id == id) {
            double totalRating = products[i].rating * products[i].ratingCount;
            products[i].ratingCount++;
            products[i].rating = (totalRating + rating) / products[i].ratingCount;
            printf("Terima kasih atas penilaian Anda!\n");
            pause();
            return;
        }
    }

    printf("Produk tidak ditemukan!\n");
    pause();
}

// ---------------- Admin - Kelola Produk (sudah ada) ----------------
void addProduct() {
    clearScreen();
    printf("=== TAMBAH PRODUK BARU ===\n\n");

    if (productCount >= MAX_PRODUCTS) {
        printf("Database produk penuh!\n");
        pause();
        return;
    }

    Product newProduct;
    newProduct.id = (productCount > 0) ? products[productCount-1].id + 1 : 1;

    printf("Nama Produk: ");
    scanf(" %[^\n]", newProduct.name);
    printf("Kategori (women/man/kids/baby/electronics/...): ");
    scanf(" %[^\n]", newProduct.category);
    printf("Ukuran: ");
    scanf("%s", newProduct.size);
    printf("Warna: ");
    scanf("%s", newProduct.color);
    printf("Harga: ");
    scanf("%lf", &newProduct.price);
    printf("Stok Online: ");
    scanf("%d", &newProduct.stockOnline);
    printf("Stok Offline: ");
    scanf("%d", &newProduct.stockOffline);
    
    newProduct.rating = 0.0;
    newProduct.ratingCount = 0;

    products[productCount] = newProduct;
    productCount++;

    // add category to graph if needed
    addCategoryIfNotExists(newProduct.category);

    printf("\nProduk berhasil ditambahkan!\n");
    pause();
}

void editProduct() {
    clearScreen();
    printf("=== EDIT PRODUK ===\n\n");

    int id;
    printf("Masukkan ID Produk: ");
    scanf("%d", &id);

    for (int i = 0; i < productCount; i++) {
        if (products[i].id == id) {
            printf("\nData saat ini:\n");
            displayProduct(products[i]);

            printf("\nMasukkan data baru:\n");
            printf("Nama: ");
            scanf(" %[^\n]", products[i].name);
            printf("Kategori: ");
            scanf(" %[^\n]", products[i].category);
            printf("Ukuran: ");
            scanf("%s", products[i].size);
            printf("Warna: ");
            scanf("%s", products[i].color);
            printf("Harga: ");
            scanf("%lf", &products[i].price);
            printf("Stok Online: ");
            scanf("%d", &products[i].stockOnline);
            printf("Stok Offline: ");
            scanf("%d", &products[i].stockOffline);

            addCategoryIfNotExists(products[i].category);

            printf("\nProduk berhasil diupdate!\n");
            pause();
            return;
        }
    }

    printf("Produk tidak ditemukan!\n");
    pause();
}

void deleteProduct() {
    clearScreen();
    printf("=== HAPUS PRODUK ===\n\n");

    int id;
    printf("Masukkan ID Produk: ");
    scanf("%d", &id);

    for (int i = 0; i < productCount; i++) {
        if (products[i].id == id) {
            for (int j = i; j < productCount - 1; j++) {
                products[j] = products[j + 1];
            }
            productCount--;
            printf("Produk berhasil dihapus!\n");
            pause();
            return;
        }
    }

    printf("Produk tidak ditemukan!\n");
    pause();
}

void viewAllProducts() {
    clearScreen();
    printf("=== SEMUA PRODUK ===\n\n");

    for (int i = 0; i < productCount; i++) {
        displayProduct(products[i]);
    }
    pause();
}

// ---------------- SuperAdmin - Kelola User ----------------
void manageUsers() {
    clearScreen();
    printf("=== KELOLA USER ===\n\n");

    printf("Daftar User:\n");
    for (int i = 0; i < userCount; i++) {
        printf("%d. %s (Role: %s)\n", users[i].id, users[i].username, users[i].role);
    }

    printf("\n1. Tambah User\n");
    printf("2. Edit Role User\n");
    printf("3. Hapus User\n");
    printf("0. Kembali\n");
    printf("\nPilihan: ");

    int choice;
    scanf("%d", &choice);

    if (choice == 1) {
        if (userCount >= MAX_USERS) {
            printf("Database user penuh!\n");
            pause();
            return;
        }

        User newUser;
        newUser.id = userCount + 1;
        newUser.isLoggedIn = false;

        printf("\nUsername: ");
        scanf("%s", newUser.username);
        printf("Password: ");
        scanf("%s", newUser.password);
        printf("Role (superadmin/admin/pembeli): ");
        scanf("%s", newUser.role);

        users[userCount] = newUser;
        userCount++;
        printf("User berhasil ditambahkan!\n");
        pause();

    } else if (choice == 2) {
        int id;
        printf("\nMasukkan ID User: ");
        scanf("%d", &id);

        for (int i = 0; i < userCount; i++) {
            if (users[i].id == id) {
                printf("Role baru (superadmin/admin/pembeli): ");
                scanf("%s", users[i].role);
                printf("Role berhasil diupdate!\n");
                pause();
                return;
            }
        }
        printf("User tidak ditemukan!\n");
        pause();

    } else if (choice == 3) {
        int id;
        printf("\nMasukkan ID User: ");
        scanf("%d", &id);

        for (int i = 0; i < userCount; i++) {
            if (users[i].id == id) {
                for (int j = i; j < userCount - 1; j++) {
                    users[j] = users[j + 1];
                }
                userCount--;
                printf("User berhasil dihapus!\n");
                pause();
                return;
            }
        }
        printf("User tidak ditemukan!\n");
        pause();
    }
}

// Menu Contact Us
void contactUs() {
    clearScreen();
    printf("=== CONTACT US ===\n\n");
    printf("Fashion Store\n");
    printf("Alamat: Jl. Fashion No. 123, Jakarta\n");
    printf("Telepon: (021) 1234-5678\n");
    printf("Email: info@fashionstore.com\n");
    printf("WhatsApp: +62 812-3456-7890\n");
    printf("Instagram: @fashionstore\n");
    printf("\nJam Operasional:\n");
    printf("Senin - Jumat: 09.00 - 21.00\n");
    printf("Sabtu - Minggu: 10.00 - 22.00\n");
    pause();
}

// ---------------- Edit Profil Pengguna ----------------
void editProfile() {
    if (currentUserId < 0) {
        printf("Harus login terlebih dahulu.\n");
        pause();
        return;
    }
    clearScreen();
    printf("=== EDIT PROFIL ===\n\n");
    printf("Username saat ini: %s\n", users[currentUserId].username);
    printf("1. Ganti Username\n");
    printf("2. Ganti Password\n");
    printf("0. Kembali\n");
    printf("Pilihan: ");
    int ch;
    scanf("%d", &ch);
    if (ch == 1) {
        char nu[MAX_NAME];
        printf("Masukkan username baru: ");
        scanf("%s", nu);
        strcpy_safe(users[currentUserId].username, nu, MAX_NAME);
        printf("Username berhasil diupdate.\n");
        pause();
    } else if (ch == 2) {
        char np[MAX_NAME];
        printf("Masukkan password baru: ");
        scanf("%s", np);
        strcpy_safe(users[currentUserId].password, np, MAX_NAME);
        printf("Password berhasil diupdate.\n");
        pause();
    }
}

// ---------------- Filter Produk (input kategori) ----------------
void filterProductsMenu() {
    clearScreen();
    printf("=== FILTER PRODUK BERDASARKAN KATEGORI ===\n\n");
    printf("Masukkan kategori (mis: women, man, kids, baby, electronics): ");
    char cat[50];
    scanf(" %[^\n]", cat);
    displayProductsByCategory(cat);
}
   // Fungsi untuk mengalokasikan dan menampilkan produk dengan rating di atas threshold
// Menggunakan array dinamis untuk menyimpan hasil filter
void getHighRatedProducts(double minRating, Product **resultArray, int *resultCount) {
    // Hitung berapa banyak produk yang memenuhi kriteria
    int count = 0;
    for (int i = 0; i < productCount; i++) {
        if (products[i].rating >= minRating) {
            count++;
        }
    }

    // Alokasikan memori untuk array dinamis
    *resultArray = (Product*) malloc(count * sizeof(Product));
    if (!(*resultArray)) {
        printf("Gagal mengalokasikan memori!\n");
        *resultCount = 0;
        return;
    }

    // Salin produk yang memenuhi kriteria ke array dinamis
    int idx = 0;
    for (int i = 0; i < productCount; i++) {
        if (products[i].rating >= minRating) {
            (*resultArray)[idx] = products[i];
            idx++;
        }
    }

    *resultCount = count;
}

void viewHighRatedProducts() {
    clearScreen();
    printf("=== PRODUK DENGAN RATING TINGGI ===\n\n");

    double minRating;
    printf("Masukkan rating minimum (1.0 - 5.0): ");
    scanf("%lf", &minRating);

    if (minRating < 1.0 || minRating > 5.0) {
        printf("Rating harus antara 1.0 dan 5.0!\n");
        pause();
        return;
    }

    Product *highRatedProducts = NULL;
    int resultCount = 0;

    // Panggil fungsi dengan array dinamis
    getHighRatedProducts(minRating, &highRatedProducts, &resultCount);

    if (resultCount == 0) {
        printf("Tidak ada produk dengan rating >= %.1f\n", minRating);
    } else {
        printf("Ditemukan %d produk dengan rating >= %.1f:\n\n", resultCount, minRating);
        for (int i = 0; i < resultCount; i++) {
            displayProduct(highRatedProducts[i]);
        }
    }

    // Bebaskan memori yang telah dialokasikan
    if (highRatedProducts != NULL) {
        free(highRatedProducts);
        highRatedProducts = NULL;
    }

    pause();
}
// ---------------- Menu untuk Pembeli / Admin / Superadmin ----------------
void customerMenu() {
    int choice;
    do {
        clearScreen();
        printf("=== FASHION STORE - PEMBELI ===\n");
        if (currentUserId >= 0) printf("User: %s\n\n", users[currentUserId].username);
        else printf("\n");
        printf("KATEGORI PRODUK:\n");
        printf("1. Women\n");
        printf("2. Man\n");
        printf("3. Kids\n");
        printf("4. Baby\n");
        printf("5. Electronics\n");
        printf("\nFITUR:\n");
        printf("6. Cari Produk\n");
        printf("7. Filter Produk (kategori lain)\n");
        printf("8. Sorting Produk\n");
        printf("9. Favorit\n");
        printf("10. Keranjang\n");
        printf("11. Konfirmasi & Checkout\n");
        printf("12. Riwayat Transaksi\n");
        printf("13. Ajukan Pengembalian Barang\n");
        printf("14. Beri Rating\n");
        printf("15. Edit Profil\n");
        printf("16. Contact Us\n");
        printf("17. Recently Viewed (Stack)\n");
        printf("18. view high rate products\n");
        printf("0. Logout\n");
        printf("\nPilihan: ");
        scanf("%d", &choice);

        switch(choice) {
            case 1: displayProductsByCategory("women"); break;
            case 2: displayProductsByCategory("man"); break;
            case 3: displayProductsByCategory("kids"); break;
            case 4: displayProductsByCategory("baby"); break;
            case 5: displayProductsByCategory("electronics"); break;
            case 6: searchProduct(); break;
            case 7: filterProductsMenu(); break;
            case 8: sortMenu(); break;
            case 9: {
                int subChoice;
                clearScreen();
                printf("1. Lihat Favorit\n2. Tambah ke Favorit\n\nPilihan: ");
                scanf("%d", &subChoice);
                if (subChoice == 1) viewFavorites();
                else if (subChoice == 2) addToFavorites();
                break;
            }
            case 10: {
                int subChoice;
                clearScreen();
                printf("1. Lihat Keranjang\n2. Tambah ke Keranjang\n\nPilihan: ");
                scanf("%d", &subChoice);
                if (subChoice == 1) viewCart();
                else if (subChoice == 2) addToCart();
                break;
            }
            case 11: checkout(); break;
            case 12: showTransactionsByUser(currentUserId); break;
            case 13: {
                clearScreen();
                printf("=== AJUKAN PENGEMBALIAN ===\n\n");
                int prodId, qty;
                printf("Masukkan Product ID: ");
                scanf("%d", &prodId);
                printf("Jumlah yg dikembalikan: ");
                scanf("%d", &qty);
                ReturnRequest r;
                r.requestId = nextReturnId++;
                r.userId = currentUserId;
                r.productId = prodId;
                r.quantity = qty;
                r.timestamp = time(NULL);
                if (enqueueReturn(r)) printf("Pengajuan pengembalian berhasil, RequestID: %d\n", r.requestId);
                else printf("Queue pengembalian penuh.\n");
                pause();
                break;
            }
            case 14: rateProduct(); break;
            case 15: editProfile(); break;
            case 16: contactUs(); break;
            case 17: showViewedStack(); break;
            case 18: viewHighRatedProducts(); break;
            case 0: logout(); break;
            default: printf("Pilihan tidak valid!\n"); pause();
        }
    } while(choice != 0);
}

void adminMenu() {
    int choice;
    do {
        clearScreen();
        printf("=== FASHION STORE - ADMIN ===\n");
        if (currentUserId >= 0) printf("User: %s\n\n", users[currentUserId].username);
        else printf("\n");
        printf("1. Lihat Semua Produk\n");
        printf("2. Tambah Produk\n");
        printf("3. Edit Produk\n");
        printf("4. Hapus Produk\n");
        printf("5. Cari Produk\n");
        printf("6. Contact Us\n");
        printf("7. Lihat Antrean Order (Queue)\n");
        printf("8. Lihat Antrean Pengembalian\n");
        printf("9. Lihat Graf Kategori\n");
        printf("10. Lihat Riwayat Login\n");
        printf("0. Logout\n");
        printf("\nPilihan: ");
        scanf("%d", &choice);

        switch(choice) {
            case 1: viewAllProducts(); break;
            case 2: addProduct(); break;
            case 3: editProduct(); break;
            case 4: deleteProduct(); break;
            case 5: searchProduct(); break;
            case 6: contactUs(); break;
            case 7: viewOrderQueue(); break;
            case 8: viewReturnQueue(); break;
            case 9: showCategoryGraph(); break;
            case 10: viewLoginHistory(); break;
            case 0: logout(); break;
            default: printf("Pilihan tidak valid!\n"); pause();
        }
    } while(choice != 0);
}

void superAdminMenu() {
    int choice;
    do {
        clearScreen();
        printf("=== FASHION STORE - SUPER ADMIN ===\n");
        if (currentUserId >= 0) printf("User: %s\n\n", users[currentUserId].username);
        else printf("\n");
        printf("1. Lihat Semua Produk\n");
        printf("2. Tambah Produk\n");
        printf("3. Edit Produk\n");
        printf("4. Hapus Produk\n");
        printf("5. Kelola User\n");
        printf("6. Cari Produk\n");
        printf("7. Contact Us\n");
        printf("8. Lihat Antrean Order (Queue)\n");
        printf("9. Lihat Antrean Pengembalian\n");
        printf("10. Lihat Graf Kategori\n");
        printf("11. Lihat Riwayat Login\n");
        printf("0. Logout\n");
        printf("\nPilihan: ");
        scanf("%d", &choice);

        switch(choice) {
            case 1: viewAllProducts(); break;
            case 2: addProduct(); break;
            case 3: editProduct(); break;
            case 4: deleteProduct(); break;
            case 5: manageUsers(); break;
            case 6: searchProduct(); break;
            case 7: contactUs(); break;
            case 8: viewOrderQueue(); break;
            case 9: viewReturnQueue(); break;
            case 10: showCategoryGraph(); break;
            case 11: viewLoginHistory(); break;
            case 0: logout(); break;
            default: printf("Pilihan tidak valid!\n"); pause();
        }
    } while(choice != 0);
}

 

// Main Program
int main() {
    initializeData();

    int choice;
    do {
        clearScreen();
        printf("======================================\n");
        printf("   SELAMAT DATANG DI FASHION STORE   \n");
        printf("======================================\n\n");
        printf("1. Login\n");
        printf("2. Contact Us\n");
        printf("0. Keluar\n");
        printf("\nPilihan: ");
        scanf("%d", &choice);

        if (choice == 1) {
            login();
            if (currentUserId >= 0) {
                if (strcmp_custom(users[currentUserId].role, "superadmin") == 0) {
                    superAdminMenu();
                } else if (strcmp_custom(users[currentUserId].role, "admin") == 0) {
                    adminMenu();
                } else if (strcmp_custom(users[currentUserId].role, "pembeli") == 0) {
                    customerMenu();
                }
            }
        } else if (choice == 2) {
            contactUs();
        } else if (choice == 0) {
            printf("\nTerima kasih telah menggunakan Fashion Store!\n");
        } else {
            printf("\nPilihan tidak valid!\n");
            pause();
        }
    } while(choice != 0);

    return 0;
}
