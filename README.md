#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>

#ifdef __unix__
#include <sys/resource.h>
#endif

#define RUNS 5
#define MAX_SIZE 100000

typedef void (*SortFunction)(int[], int);

typedef struct {
    const char *name;
    SortFunction function;
} Algorithm;

typedef struct {
    double average_ms;
    double peak_memory_kb;
} BenchmarkResult;

void bubbleSort(int arr[], int n) {
    int i, j, temp, swapped;
    for (i = 0; i < n - 1; i++) {
        swapped = 0;
        for (j = 0; j < n - i - 1; j++) {
            if (arr[j] > arr[j + 1]) {
                temp = arr[j];
                arr[j] = arr[j + 1];
                arr[j + 1] = temp;
                swapped = 1;
            }
        }
        if (!swapped) break;
    }
}

void selectionSort(int arr[], int n) {
    int i, j, min_index, temp;
    for (i = 0; i < n - 1; i++) {
        min_index = i;
        for (j = i + 1; j < n; j++) {
            if (arr[j] < arr[min_index]) min_index = j;
        }
        temp = arr[i];
        arr[i] = arr[min_index];
        arr[min_index] = temp;
    }
}

void insertionSort(int arr[], int n) {
    int i, j, key;
    for (i = 1; i < n; i++) {
        key = arr[i];
        j = i - 1;
        while (j >= 0 && arr[j] > key) {
            arr[j + 1] = arr[j];
            j--;
        }
        arr[j + 1] = key;
    }
}

void merge(int arr[], int left, int mid, int right) {
    int n1 = mid - left + 1;
    int n2 = right - mid;
    int *left_array = (int *)malloc((size_t)n1 * sizeof(int));
    int *right_array = (int *)malloc((size_t)n2 * sizeof(int));
    int i, j, k;

    if (left_array == NULL || right_array == NULL) {
        fprintf(stderr, "Memory allocation failed.\n");
        free(left_array);
        free(right_array);
        exit(EXIT_FAILURE);
    }

    for (i = 0; i < n1; i++) left_array[i] = arr[left + i];
    for (j = 0; j < n2; j++) right_array[j] = arr[mid + 1 + j];

    i = 0;
    j = 0;
    k = left;

    while (i < n1 && j < n2) {
        if (left_array[i] <= right_array[j]) arr[k++] = left_array[i++];
        else arr[k++] = right_array[j++];
    }
    while (i < n1) arr[k++] = left_array[i++];
    while (j < n2) arr[k++] = right_array[j++];

    free(left_array);
    free(right_array);
}

void mergeSortRecursive(int arr[], int left, int right) {
    if (left < right) {
        int mid = left + (right - left) / 2;
        mergeSortRecursive(arr, left, mid);
        mergeSortRecursive(arr, mid + 1, right);
        merge(arr, left, mid, right);
    }
}

void mergeSort(int arr[], int n) {
    if (n > 1) mergeSortRecursive(arr, 0, n - 1);
}

/* Quick Sort uses the last element as the pivot. */
int partitionArray(int arr[], int low, int high) {
    int pivot = arr[high];
    int i = low - 1;
    int j, temp;

    for (j = low; j < high; j++) {
        if (arr[j] <= pivot) {
            i++;
            temp = arr[i];
            arr[i] = arr[j];
            arr[j] = temp;
        }
    }

    temp = arr[i + 1];
    arr[i + 1] = arr[high];
    arr[high] = temp;
    return i + 1;
}

void quickSortRecursive(int arr[], int low, int high) {
    if (low < high) {
        int pivot_index = partitionArray(arr, low, high);
        quickSortRecursive(arr, low, pivot_index - 1);
        quickSortRecursive(arr, pivot_index + 1, high);
    }
}

void quickSort(int arr[], int n) {
    if (n > 1) quickSortRecursive(arr, 0, n - 1);
}

Algorithm algorithms[] = {
    {"Bubble Sort", bubbleSort},
    {"Selection Sort", selectionSort},
    {"Insertion Sort", insertionSort},
    {"Merge Sort", mergeSort},
    {"Quick Sort", quickSort}
};

const int ALGORITHM_COUNT = 5;

void generateRandomArray(int arr[], int n) {
    int i;
    for (i = 0; i < n; i++) arr[i] = rand() % 1000000;
}

int *copyArray(const int source[], int n) {
    int *copy = (int *)malloc((size_t)n * sizeof(int));
    if (copy == NULL) {
        fprintf(stderr, "Memory allocation failed.\n");
        exit(EXIT_FAILURE);
    }
    memcpy(copy, source, (size_t)n * sizeof(int));
    return copy;
}

int isSorted(const int arr[], int n) {
    int i;
    for (i = 1; i < n; i++) {
        if (arr[i - 1] > arr[i]) return 0;
    }
    return 1;
}

double getPeakMemoryKB(void) {
#ifdef __unix__
    struct rusage usage;
    if (getrusage(RUSAGE_SELF, &usage) == 0) {
#ifdef __APPLE__
        return (double)usage.ru_maxrss / 1024.0;
#else
        return (double)usage.ru_maxrss;
#endif
    }
#endif
    return -1.0;
}

BenchmarkResult benchmark(const Algorithm *algorithm, const int original[], int n) {
    BenchmarkResult result;
    double total_ms = 0.0;
    double peak_kb = -1.0;
    int run;

    for (run = 0; run < RUNS; run++) {
        int *work = copyArray(original, n);
        clock_t start = clock();
        algorithm->function(work, n);
        clock_t finish = clock();

        total_ms += ((double)(finish - start) / CLOCKS_PER_SEC) * 1000.0;

        if (!isSorted(work, n)) {
            fprintf(stderr, "Error: %s did not sort the array correctly.\n", algorithm->name);
            free(work);
            exit(EXIT_FAILURE);
        }

        {
            double measured = getPeakMemoryKB();
            if (measured > peak_kb) peak_kb = measured;
        }
        free(work);
    }

    result.average_ms = total_ms / RUNS;
    result.peak_memory_kb = peak_kb;
    return result;
}

void printArray(const int arr[], int n) {
    int i;
    int limit = n < 30 ? n : 30;
    for (i = 0; i < limit; i++) printf("%d ", arr[i]);
    if (n > limit) printf("...");
    printf("\n");
}

int readPositiveInt(const char *prompt, int min_value, int max_value) {
    char line[100];
    int value;
    char extra;

    while (1) {
        printf("%s", prompt);
        if (fgets(line, sizeof(line), stdin) == NULL) return -1;
        if (sscanf(line, "%d %c", &value, &extra) == 1 &&
            value >= min_value && value <= max_value) {
            return value;
        }
        printf("Invalid input. Enter a number between %d and %d.\n", min_value, max_value);
    }
}

void runOneAlgorithm(int index) {
    int n, *data;
    BenchmarkResult result;

    n = readPositiveInt("Enter array size (1 - 100000): ", 1, MAX_SIZE);
    if (n == -1) return;

    data = (int *)malloc((size_t)n * sizeof(int));
    if (data == NULL) {
        printf("Memory allocation failed.\n");
        return;
    }

    generateRandomArray(data, n);
    printf("\nOriginal array (first 30 values): ");
    printArray(data, n);

    result = benchmark(&algorithms[index], data, n);

    algorithms[index].function(data, n);
    printf("Sorted array (first 30 values):   ");
    printArray(data, n);

    printf("\nAlgorithm: %s\n", algorithms[index].name);
    printf("Average execution time (%d runs): %.3f ms\n", RUNS, result.average_ms);
    if (result.peak_memory_kb >= 0)
        printf("Process peak memory: %.2f KB\n", result.peak_memory_kb);
    else
        printf("Peak memory measurement is not available on this platform.\n");

    free(data);
}

void writeCsvHeader(FILE *file) {
    fprintf(file, "Algorithm,Array Size,Average Time (ms),Process Peak Memory (KB),Runs\n");
}

void compareAll(void) {
    const int sizes[] = {1000, 5000, 10000, 50000, 100000};
    const int size_count = (int)(sizeof(sizes) / sizeof(sizes[0]));
    int s, a;
    FILE *csv = fopen("results/benchmark_results.csv", "w");

    if (csv == NULL) {
        csv = fopen("benchmark_results.csv", "w");
    }
    if (csv == NULL) {
        printf("Could not create the CSV results file.\n");
        return;
    }

    writeCsvHeader(csv);
    printf("\n%-16s %-12s %-18s %-20s\n", "Algorithm", "Array Size", "Average Time (ms)", "Peak Memory (KB)");
    printf("--------------------------------------------------------------------------\n");

    for (s = 0; s < size_count; s++) {
        int n = sizes[s];
        int *original = (int *)malloc((size_t)n * sizeof(int));
        if (original == NULL) {
            printf("Memory allocation failed.\n");
            fclose(csv);
            return;
        }
        generateRandomArray(original, n);

        for (a = 0; a < ALGORITHM_COUNT; a++) {
            /* Quadratic algorithms are limited to 10,000 elements. */
            if (n > 10000 && a < 3) continue;

            {
                BenchmarkResult result = benchmark(&algorithms[a], original, n);
                printf("%-16s %-12d %-18.3f ", algorithms[a].name, n, result.average_ms);
                if (result.peak_memory_kb >= 0)
                    printf("%-20.2f\n", result.peak_memory_kb);
                else
                    printf("%-20s\n", "N/A");

                fprintf(csv, "%s,%d,%.3f,%.2f,%d\n",
                        algorithms[a].name, n, result.average_ms,
                        result.peak_memory_kb, RUNS);
            }
        }
        free(original);
    }

    fclose(csv);
    printf("\nComparison completed. Results saved in results/benchmark_results.csv\n");
    printf("Bubble, Selection and Insertion Sort are tested up to 10,000 elements.\n");
    printf("Merge Sort and Quick Sort are tested up to 100,000 elements.\n");
}

void runBigArrayExample(void) {
    int n = 100000;
    int *data = (int *)malloc((size_t)n * sizeof(int));
    int i;

    if (data == NULL) {
        printf("Memory allocation failed.\n");
        return;
    }

    generateRandomArray(data, n);
    printf("\nBig array example: %d random integers (fixed seed).\n", n);

    for (i = 3; i < 5; i++) {
        BenchmarkResult result = benchmark(&algorithms[i], data, n);
        printf("%s: %.3f ms, process peak memory: ", algorithms[i].name, result.average_ms);
        if (result.peak_memory_kb >= 0) printf("%.2f KB\n", result.peak_memory_kb);
        else printf("N/A\n");
    }

    free(data);
}

int main(void) {
    int choice;
    srand(2026);

    do {
        printf("\n========== SORTING ALGORITHMS ==========\n");
        printf("1. Bubble Sort\n");
        printf("2. Selection Sort\n");
        printf("3. Insertion Sort\n");
        printf("4. Merge Sort\n");
        printf("5. Quick Sort\n");
        printf("6. Compare all algorithms\n");
        printf("7. Big array example (100,000 elements)\n");
        printf("0. Exit\n");

        choice = readPositiveInt("Your choice: ", 0, 7);
        if (choice == -1) break;

        if (choice >= 1 && choice <= 5) {
            runOneAlgorithm(choice - 1);
        } else if (choice == 6) {
            compareAll();
        } else if (choice == 7) {
            runBigArrayExample();
        } else if (choice == 0) {
            printf("Goodbye!\n");
        }
    } while (choice != 0);

    return 0;
}
