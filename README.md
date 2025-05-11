# Multithread-Simulator
### Output 
![image](https://github.com/user-attachments/assets/7322f795-90b4-45cd-a402-d3b12da02088)
Program
```
import threading
import time
import random
import sys

class Memory:
    def __init__(self):
        self.data = {}

    def read(self, address):
        return self.data.get(address, 0)

    def write(self, address, value):
        self.data[address] = value

class Cache:
    def __init__(self, core_id, memory):
        self.core_id = core_id
        self.memory = memory
        self.data = {}
        self.state = {}
        self.msg_count = 0
        self.read_time = 0  # Total time spent reading
        self.write_time = 0  # Total time spent writing
        self.log = []  # FIX: Logging untuk setiap cache

    def read(self, address, use_coherence, all_caches):
        start_time = time.time()
        if use_coherence and self.state.get(address) == 'Invalid':
            self.fetch_from_memory(address)
        self.read_time += time.time() - start_time
        val = self.data.get(address, 0)
        self.log.append(f"[Core {self.core_id}] READ {address} => {val}")
        return val

    def write(self, address, value, use_coherence, all_caches):
        start_time = time.time()
        if use_coherence:
            self.invalidate_others(address, all_caches)
            self.state[address] = 'Modified'
        self.data[address] = value
        self.memory.write(address, value)
        self.write_time += time.time() - start_time
        self.log.append(f"[Core {self.core_id}] WRITE {address} <= {value}")

    def fetch_from_memory(self, address):
        self.data[address] = self.memory.read(address)
        self.state[address] = 'Shared'
        self.msg_count += 1
        self.log.append(f"[Core {self.core_id}] FETCH from memory: {address}")

    def invalidate_others(self, address, all_caches):
        for cache in all_caches:
            if cache.core_id != self.core_id and address in cache.state and cache.state[address] != 'Invalid':
                cache.state[address] = 'Invalid'
                self.msg_count += 1
                self.log.append(f"[Core {self.core_id}] INVALIDATE address {address} in Core {cache.core_id}")

class CoreThread(threading.Thread):
    def __init__(self, core_id, use_coherence, caches, num_accesses):
        super().__init__()
        self.core_id = core_id
        self.cache = caches[core_id]
        self.use_coherence = use_coherence
        self.all_caches = caches
        self.num_accesses = num_accesses

    def run(self):
        for _ in range(self.num_accesses):
            op = random.choice(['read', 'write'])
            address = random.choice(['x', 'y'])
            if op == 'read':
                self.cache.read(address, self.use_coherence, self.all_caches)
            else:
                val = random.randint(0, 100)
                self.cache.write(address, val, self.use_coherence, self.all_caches)

def simulate(use_coherence, num_cores, num_accesses):
    memory = Memory()
    caches = [Cache(i, memory) for i in range(num_cores)]
    threads = [CoreThread(i, use_coherence, caches, num_accesses) for i in range(num_cores)]

    start_time = time.time()
    for t in threads:
        t.start()
    for t in threads:
        t.join()
    end_time = time.time()

    total_msgs = sum(cache.msg_count for cache in caches)
    total_read_time = sum(cache.read_time for cache in caches)
    total_write_time = sum(cache.write_time for cache in caches)

    return round(end_time - start_time, 6), total_msgs, total_read_time, total_write_time, caches

def analyze_performance(time_no, msgs_no, read_time_no, write_time_no, time_yes, msgs_yes, read_time_yes, write_time_yes, num_cores):
    time_efficiency = (time_no - time_yes) / time_no * 100 if time_no > 0 else 0
    msg_efficiency = (msgs_no - msgs_yes) / msgs_yes * 100 if msgs_yes > 0 else 0

    avg_time_no = time_no / num_cores
    avg_time_yes = time_yes / num_cores

    avg_msgs_no = msgs_no / num_cores
    avg_msgs_yes = msgs_yes / num_cores

    avg_read_time_no = read_time_no / num_cores
    avg_read_time_yes = read_time_yes / num_cores

    avg_write_time_no = write_time_no / num_cores
    avg_write_time_yes = write_time_yes / num_cores

    return time_efficiency, msg_efficiency, avg_time_no, avg_time_yes, avg_msgs_no, avg_msgs_yes, avg_read_time_no, avg_read_time_yes, avg_write_time_no, avg_write_time_yes

def save_log_to_file(caches, filename='log_akses_cache.txt'):
    with open(filename, 'w') as f:
        for cache in caches:
            f.write(f"=== Core {cache.core_id} ===\n")
            for entry in cache.log:
                f.write(entry + '\n')
            f.write("\n")

def main():
    random.seed(42)
    try:
        num_cores = int(input("Masukkan jumlah core (misal 4): "))
        num_accesses = int(input("Masukkan jumlah akses per core (misal 100): "))
    except ValueError:
        print("Input tidak valid. Harus berupa angka.")
        sys.exit(1)

    print("\n[1] Menjalankan simulasi TANPA protokol koherensi...")
    time_no, msgs_no, read_time_no, write_time_no, caches_no = simulate(False, num_cores, num_accesses)

    print("[2] Menjalankan simulasi DENGAN protokol koherensi...")
    time_yes, msgs_yes, read_time_yes, write_time_yes, caches_yes = simulate(True, num_cores, num_accesses)

    print("\n=== HASIL PERBANDINGAN ===")
    print(f"{'Mode':<25}{'Waktu (s)':<12}{'Pesan Koherensi':<18}{'Rata-rata Waktu per Core (s)':<25}")
    print("-" * 70)
    print(f"{'Tanpa Koherensi':<25}{time_no:<12}{msgs_no:<18}{time_no/num_cores:.6f}")
    print(f"{'Dengan Koherensi':<25}{time_yes:<12}{msgs_yes:<18}{time_yes/num_cores:.6f}")
    print("-" * 70)

    perf = analyze_performance(
        time_no, msgs_no, read_time_no, write_time_no,
        time_yes, msgs_yes, read_time_yes, write_time_yes, num_cores
    )

    print(f"\nEfisiensi Waktu: {perf[0]:.2f}%")
    print(f"Efisiensi Pesan Koherensi: {perf[1]:.2f}%")
    print(f"Rata-rata pesan koherensi per core (tanpa koherensi): {perf[4]:.2f}")
    print(f"Rata-rata pesan koherensi per core (dengan koherensi): {perf[5]:.2f}")
    print(f"Rata-rata waktu baca per core (tanpa koherensi): {perf[6]:.6f} detik")
    print(f"Rata-rata waktu baca per core (dengan koherensi): {perf[7]:.6f} detik")
    print(f"Rata-rata waktu tulis per core (tanpa koherensi): {perf[8]:.6f} detik")
    print(f"Rata-rata waktu tulis per core (dengan koherensi): {perf[9]:.6f} detik")

    save_log_to_file(caches_yes)

if __name__ == '__main__':
    main()


```
