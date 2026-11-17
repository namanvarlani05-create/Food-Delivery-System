/*
 * Project 55: Food Delivery Order and Tracking System
 * ---------------------------------------------------
 * A modular, menu-driven order management simulation in C.
 *
 * Concepts used:
 *  - Structures and arrays of structures (Order, OrderList)
 *  - Pointers and dynamic memory (malloc / realloc / free)
 *  - Loops and decision-making (status flow, delay / bonus logic)
 *  - Functions (modular design)
 *  - File handling (save / load orders)
 *
 * Compile:  gcc food_delivery.c -o food_delivery
 * Run:      ./food_delivery
 */

#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#define NAME_LEN        50
#define INITIAL_CAP     4
#define DATA_FILE       "orders.dat"

/* Delivery performance rules (minutes / rupees) */
#define BONUS_THRESHOLD 10   /* delivered this many minutes early => bonus   */
#define BONUS_AMOUNT    50.0f
#define LATE_THRESHOLD  15   /* delivered this many minutes late  => penalty */
#define PENALTY_AMOUNT  30.0f

/* ---------- Order status (the order flow) ---------- */
typedef enum {
    PLACED = 0,
    PREPARING,
    OUT_FOR_DELIVERY,
    DELIVERED,
    CANCELLED
} Status;

static const char *STATUS_TEXT[] = {
    "Placed", "Preparing", "Out for Delivery", "Delivered", "Cancelled"
};

/* ---------- Data structures ---------- */
typedef struct {
    int    order_id;
    char   customer_name[NAME_LEN];
    char   restaurant[NAME_LEN];
    Status status;
    int    delivery_time;   /* actual delivery time in minutes (0 until delivered) */
    int    expected_time;   /* promised delivery time in minutes                    */
    float  amount;          /* bill amount, used for revenue                        */
} Order;

typedef struct {
    Order *orders;   /* dynamically allocated array of orders */
    int    count;
    int    capacity;
} OrderList;

/* ---------- Input helpers ---------- */
static void read_line(const char *prompt, char *buf, int size) {
    printf("%s", prompt);
    if (fgets(buf, size, stdin) == NULL) {
        buf[0] = '\0';
        return;
    }
    buf[strcspn(buf, "\n")] = '\0';
}

static int read_int(const char *prompt) {
    char line[64];
    int value;
    while (1) {
        printf("%s", prompt);
        if (fgets(line, sizeof line, stdin) == NULL) return 0;
        if (sscanf(line, "%d", &value) == 1) return value;
        printf("  Invalid number, try again.\n");
    }
}

static float read_float(const char *prompt) {
    char line[64];
    float value;
    while (1) {
        printf("%s", prompt);
        if (fgets(line, sizeof line, stdin) == NULL) return 0;
        if (sscanf(line, "%f", &value) == 1) return value;
        printf("  Invalid number, try again.\n");
    }
}

/* ---------- List management (pointers + dynamic memory) ---------- */
static int init_list(OrderList *list) {
    list->orders = (Order *)malloc(INITIAL_CAP * sizeof(Order));
    if (list->orders == NULL) return 0;
    list->count = 0;
    list->capacity = INITIAL_CAP;
    return 1;
}

static int grow_list(OrderList *list) {
    int new_cap = list->capacity * 2;
    Order *tmp = (Order *)realloc(list->orders, new_cap * sizeof(Order));
    if (tmp == NULL) return 0;
    list->orders = tmp;
    list->capacity = new_cap;
    return 1;
}

static void free_list(OrderList *list) {
    free(list->orders);
    list->orders = NULL;
    list->count = list->capacity = 0;
}

/* Returns pointer to the order with the given id, or NULL. */
static Order *find_order(OrderList *list, int id) {
    Order *p;
    for (p = list->orders; p < list->orders + list->count; p++) {
        if (p->order_id == id) return p;
    }
    return NULL;
}

/* ---------- Core features ---------- */
static void add_order(OrderList *list) {
    Order o;

    if (list->count == list->capacity && !grow_list(list)) {
        printf("Memory allocation failed.\n");
        return;
    }

    o.order_id = read_int("Order ID: ");
    if (o.order_id <= 0) {
        printf("Order ID must be positive.\n");
        return;
    }
    if (find_order(list, o.order_id) != NULL) {
        printf("An order with ID %d already exists.\n", o.order_id);
        return;
    }

    read_line("Customer name: ", o.customer_name, NAME_LEN);
    read_line("Restaurant: ", o.restaurant, NAME_LEN);
    if (o.customer_name[0] == '\0' || o.restaurant[0] == '\0') {
        printf("Name and restaurant cannot be empty.\n");
        return;
    }

    o.expected_time = read_int("Expected delivery time (minutes): ");
    o.amount = read_float("Order amount (Rs.): ");
    if (o.expected_time <= 0 || o.amount < 0) {
        printf("Invalid expected time or amount.\n");
        return;
    }

    o.status = PLACED;
    o.delivery_time = 0;

    list->orders[list->count++] = o;
    printf("Order %d placed successfully.\n", o.order_id);
}

/* Decide whether moving from `cur` to `next` is allowed. */
static int is_valid_transition(Status cur, Status next) {
    if (cur == DELIVERED || cur == CANCELLED) return 0;   /* final states */
    if (next == CANCELLED) return cur != OUT_FOR_DELIVERY; /* too late to cancel */
    return next == cur + 1;                                /* must go step by step */
}

static void print_delay_or_bonus(const Order *o) {
    int diff = o->delivery_time - o->expected_time;   /* >0 means late */

    if (diff <= -BONUS_THRESHOLD)
        printf("  Early by %d min -> Delivery partner bonus: Rs.%.2f\n",
               -diff, BONUS_AMOUNT);
    else if (diff > LATE_THRESHOLD)
        printf("  Late by %d min -> Penalty applied: Rs.%.2f\n",
               diff, PENALTY_AMOUNT);
    else if (diff > 0)
        printf("  Slightly late by %d min -> No penalty.\n", diff);
    else
        printf("  Delivered on time.\n");
}

static void update_status(OrderList *list) {
    int id = read_int("Order ID to update: ");
    Order *o = find_order(list, id);
    int choice, i;

    if (o == NULL) {
        printf("Order %d not found.\n", id);
        return;
    }

    printf("Current status: %s\n", STATUS_TEXT[o->status]);
    printf("Select new status:\n");
    for (i = 0; i < 5; i++)
        printf("  %d. %s\n", i + 1, STATUS_TEXT[i]);
    choice = read_int("Choice: ");

    if (choice < 1 || choice > 5) {
        printf("Invalid choice.\n");
        return;
    }

    Status next = (Status)(choice - 1);
    if (!is_valid_transition(o->status, next)) {
        printf("Cannot change from '%s' to '%s'.\n",
               STATUS_TEXT[o->status], STATUS_TEXT[next]);
        return;
    }

    if (next == DELIVERED) {
        o->delivery_time = read_int("Actual delivery time (minutes): ");
        if (o->delivery_time <= 0) {
            printf("Invalid delivery time. Status not changed.\n");
            o->delivery_time = 0;
            return;
        }
        o->status = DELIVERED;
        printf("Order %d delivered.\n", o->order_id);
        print_delay_or_bonus(o);
    } else {
        o->status = next;
        printf("Order %d is now '%s'.\n", o->order_id, STATUS_TEXT[next]);
    }
}

static void print_header(void) {
    printf("%-6s %-18s %-18s %-17s %-9s %-9s %-9s\n",
           "ID", "Customer", "Restaurant", "Status",
           "Expected", "Actual", "Amount");
    printf("--------------------------------------------------------------------------------------\n");
}

static void print_order(const Order *o) {
    char actual[16];
    if (o->status == DELIVERED) snprintf(actual, sizeof actual, "%d min", o->delivery_time);
    else                        snprintf(actual, sizeof actual, "-");

    printf("%-6d %-18s %-18s %-17s %-3d min   %-9s %.2f\n",
           o->order_id, o->customer_name, o->restaurant,
           STATUS_TEXT[o->status], o->expected_time, actual, o->amount);
}

static void view_all_orders(const OrderList *list) {
    int i;
    if (list->count == 0) {
        printf("No orders yet.\n");
        return;
    }
    print_header();
    for (i = 0; i < list->count; i++)
        print_order(&list->orders[i]);
}

static void view_delivery_history(const OrderList *list) {
    int i, found = 0;
    for (i = 0; i < list->count; i++) {
        const Order *o = &list->orders[i];
        if (o->status == DELIVERED || o->status == CANCELLED) {
            if (!found) {
                print_header();
                found = 1;
            }
            print_order(o);
        }
    }
    if (!found) printf("No completed deliveries yet.\n");
}

static void track_order(OrderList *list) {
    int id = read_int("Order ID to track: ");
    Order *o = find_order(list, id);
    if (o == NULL) {
        printf("Order %d not found.\n", id);
        return;
    }
    print_header();
    print_order(o);
}

static void generate_report(const OrderList *list) {
    int i, delivered = 0, cancelled = 0, active = 0;
    int total_time = 0, on_time = 0, late = 0, early_bonus = 0;
    float revenue = 0.0f, bonus_total = 0.0f, penalty_total = 0.0f;

    for (i = 0; i < list->count; i++) {
        const Order *o = &list->orders[i];
        switch (o->status) {
            case DELIVERED: {
                int diff = o->delivery_time - o->expected_time;
                delivered++;
                total_time += o->delivery_time;
                revenue += o->amount;
                if (diff <= -BONUS_THRESHOLD) { early_bonus++; bonus_total += BONUS_AMOUNT; }
                if (diff > LATE_THRESHOLD)    { penalty_total += PENALTY_AMOUNT; }
                if (diff > 0) late++; else on_time++;
                break;
            }
            case CANCELLED:
                cancelled++;
                break;
            default:
                active++;
        }
    }

    printf("\n========== PERFORMANCE REPORT ==========\n");
    printf("Total orders        : %d\n", list->count);
    printf("Active orders       : %d\n", active);
    printf("Delivered           : %d\n", delivered);
    printf("Cancelled           : %d\n", cancelled);
    if (delivered > 0) {
        printf("Average delivery    : %.2f minutes\n", (float)total_time / delivered);
        printf("On time / early     : %d\n", on_time);
        printf("Late                : %d\n", late);
        printf("Bonuses paid        : Rs.%.2f (%d orders)\n", bonus_total, early_bonus);
        printf("Penalties           : Rs.%.2f\n", penalty_total);
    } else {
        printf("Average delivery    : N/A (no deliveries yet)\n");
    }
    printf("Total revenue       : Rs.%.2f\n", revenue);
    printf("========================================\n");
}

/* ---------- File handling ---------- */
static void save_orders(const OrderList *list) {
    FILE *fp = fopen(DATA_FILE, "wb");
    if (fp == NULL) {
        printf("Could not open %s for writing.\n", DATA_FILE);
        return;
    }
    fwrite(&list->count, sizeof(int), 1, fp);
    fwrite(list->orders, sizeof(Order), list->count, fp);
    fclose(fp);
    printf("Saved %d order(s) to %s.\n", list->count, DATA_FILE);
}

static void load_orders(OrderList *list, int quiet) {
    FILE *fp = fopen(DATA_FILE, "rb");
    int n;

    if (fp == NULL) {
        if (!quiet) printf("No saved data found.\n");
        return;
    }
    if (fread(&n, sizeof(int), 1, fp) != 1 || n < 0) {
        printf("Data file is corrupted.\n");
        fclose(fp);
        return;
    }
    while (n > list->capacity) {
        if (!grow_list(list)) {
            printf("Memory allocation failed.\n");
            fclose(fp);
            return;
        }
    }
    list->count = (int)fread(list->orders, sizeof(Order), n, fp);
    fclose(fp);
    if (!quiet || list->count > 0)
        printf("Loaded %d order(s) from %s.\n", list->count, DATA_FILE);
}

/* ---------- Menu ---------- */
static void show_menu(void) {
    printf("\n===== FOOD DELIVERY TRACKER =====\n");
    printf(" 1. Add new order\n");
    printf(" 2. Update order status\n");
    printf(" 3. Track an order\n");
    printf(" 4. View all orders\n");
    printf(" 5. View delivery history\n");
    printf(" 6. Performance report\n");
    printf(" 7. Save orders to file\n");
    printf(" 8. Load orders from file\n");
    printf(" 0. Exit\n");
}

int main(void) {
    OrderList list;
    int choice;

    if (!init_list(&list)) {
        printf("Memory allocation failed.\n");
        return 1;
    }
    load_orders(&list, 1);   /* auto-load previous session if present */

    do {
        show_menu();
        choice = read_int("Enter choice: ");
        switch (choice) {
            case 1: add_order(&list);             break;
            case 2: update_status(&list);         break;
            case 3: track_order(&list);           break;
            case 4: view_all_orders(&list);       break;
            case 5: view_delivery_history(&list); break;
            case 6: generate_report(&list);       break;
            case 7: save_orders(&list);           break;
            case 8: load_orders(&list, 0);        break;
            case 0:
                save_orders(&list);
                printf("Goodbye!\n");
                break;
            default: printf("Invalid choice.\n");
        }
    } while (choice != 0);

    free_list(&list);
    return 0;
}
