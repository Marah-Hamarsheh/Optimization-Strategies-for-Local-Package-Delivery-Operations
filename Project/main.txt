# Marah Hamarsheh 1220281
# Joud Thaher 1221381

import os
import time
import math
import random
import copy
from typing import List, Tuple
import tkinter as tk
from matplotlib.backends.backend_tkagg import FigureCanvasTkAgg
import matplotlib.pyplot as plt


class Package:
    def __init__(self, id: int, x: float, y: float, weight: float, priority: int):
        self.id = id
        self.x = x
        self.y = y
        self.weight = weight
        self.priority = priority

    def __str__(self):
        return f"Package #{self.id}: Loc ({self.x:.1f}, {self.y:.1f}), Weight: {self.weight:.1f}kg, Priority: {self.priority}"


class Vehicle:
    def __init__(self, id: int, capacity: float):
        self.id = id
        self.capacity = capacity
        self.packages = []

    def total_weight(self):
        return sum(p.weight for p in self.packages)

    def available_capacity(self):
        return self.capacity - self.total_weight()

    def is_overloaded(self):
        return self.total_weight() > self.capacity

    def utilization_ratio(self):
        return self.total_weight() / self.capacity if self.capacity else 0

    def __str__(self):
        return f"Vehicle #{self.id}: Cap {self.capacity:.1f}kg, Used {self.total_weight():.1f}kg, Pkgs {len(self.packages)}"


class Solution:
    def __init__(self, vehicles: List[Vehicle]):
        self.vehicles = copy.deepcopy(vehicles)
        self.fitness = float('inf')

    def deep_copy(self):
        return copy.deepcopy(self)


class DeliverySystem:
    def __init__(self):
        self.vehicles = []
        self.packages = []
        self.shop_location = (0.0, 0.0)
        self.current_algorithm = "simulated_annealing"

        # Weights for fitness calculation
        self.W_DISTANCE = 1.0
        self.W_CAPACITY = 100.0
        self.W_PRIORITY = 2.0
        self.W_BALANCE = 0.5

        self.MAX_POSSIBLE_DISTANCE = 300.0
        self.MAX_PRIORITY_SCORE = 5000.0

    def clear_screen(self):
        os.system('cls' if os.name == 'nt' else 'clear')

    def print_header(self):
        self.clear_screen()
        print("=" * 80)
        print(" " * 20 + "PACKAGE DELIVERY OPTIMIZATION SYSTEM")
        print("=" * 80)
        print()

    def calculate_distance(self, point1: Tuple[float, float], point2: Tuple[float, float]) -> float:
        return math.sqrt((point1[0] - point2[0]) ** 2 + (point1[1] - point2[1]) ** 2)

    def calculate_total_distance(self, vehicle: Vehicle) -> float:
        if not vehicle.packages:
            return 0.0

        unvisited = vehicle.packages.copy()
        current = self.shop_location
        total_distance = 0.0

        while unvisited:
            next_pkg = min(unvisited, key=lambda p: self.calculate_distance(current, (p.x, p.y)))
            total_distance += self.calculate_distance(current, (next_pkg.x, next_pkg.y))
            current = (next_pkg.x, next_pkg.y)
            unvisited.remove(next_pkg)

        total_distance += self.calculate_distance(current, self.shop_location)
        return total_distance

    def calculate_route_times(self, vehicle: Vehicle) -> List[float]:
        if not vehicle.packages:
            return []

        delivery_times = []
        cumulative_time = 0.0
        current = self.shop_location

        for package in vehicle.packages:
            destination = (package.x, package.y)
            distance = self.calculate_distance(current, destination)
            cumulative_time += distance
            delivery_times.append(cumulative_time)
            current = destination

        return delivery_times

    def calculate_standard_deviation(self, values: List[float]) -> float:
        if not values:
            return 0.0
        mean = sum(values) / len(values)
        return math.sqrt(sum((x - mean) ** 2 for x in values) / len(values))

    def evaluate_fitness(self, solution: Solution) -> float:
        if not self.validate_solution(solution):
            return float('inf')

        total_distance = 0.0
        capacity_violation = 0.0
        priority_score = 0.0
        vehicle_utilizations = []
        late_high_priority = 0

        for vehicle in solution.vehicles:
            if not vehicle.packages:
                continue

            distance = self.calculate_total_distance(vehicle)
            total_distance += distance

            if vehicle.is_overloaded():
                excess = vehicle.total_weight() - vehicle.capacity
                capacity_violation += (excess ** 2)

            delivery_times = self.calculate_route_times(vehicle)
            for i, package in enumerate(vehicle.packages):
                priority_weight = (6 - package.priority) ** 2
                priority_score += delivery_times[i] * priority_weight

                if package.priority <= 2 and delivery_times[i] > 50:
                    late_high_priority += 1

            vehicle_utilizations.append(vehicle.utilization_ratio())

        norm_distance = min(total_distance / self.MAX_POSSIBLE_DISTANCE, 2.0)
        norm_priority = min(priority_score / self.MAX_PRIORITY_SCORE, 2.0)
        utilization_balance = self.calculate_standard_deviation(vehicle_utilizations or [0])

        fitness = (
                self.W_DISTANCE * norm_distance +
                self.W_CAPACITY * capacity_violation +
                self.W_PRIORITY * norm_priority +
                self.W_BALANCE * utilization_balance +
                late_high_priority * 10
        )

        return max(fitness, 0.0001)

    def validate_solution(self, solution: Solution) -> bool:
        max_capacity = max(v.capacity for v in self.vehicles)
        assigned_ids = {p.id for v in solution.vehicles for p in v.packages}
        missing_packages = [
            p for p in self.packages
            if p.id not in assigned_ids and p.weight <= max_capacity
        ]

        if missing_packages:
            return False

        overloaded = [v for v in solution.vehicles if v.is_overloaded()]
        if overloaded:
            return False

        return True

    def create_initial_solution(self) -> Solution:
        max_capacity = max(v.capacity for v in self.vehicles)
        solution = Solution(self.vehicles)
        sorted_packages = sorted(self.packages, key=lambda p: (p.priority, -p.weight))

        for package in sorted_packages:
            if package.weight > max_capacity:
                continue

            best_vehicle = None
            best_remaining = float('inf')

            for vehicle in solution.vehicles:
                remaining = vehicle.available_capacity() - package.weight
                if remaining >= 0 and remaining < best_remaining:
                    best_vehicle = vehicle
                    best_remaining = remaining

            if best_vehicle:
                best_vehicle.packages.append(package)
            else:
                continue

        solution.fitness = self.evaluate_fitness(solution)
        return solution

    def create_random_solution(self) -> Solution:
        solution = Solution(self.vehicles)
        packages_copy = sorted(self.packages, key=lambda p: (p.priority, -p.weight))
        random.shuffle(packages_copy)
        max_capacity = max(v.capacity for v in solution.vehicles)

        for package in packages_copy:
            if package.weight > max_capacity:
                continue

            possible_vehicles = [v for v in solution.vehicles
                               if v.available_capacity() >= package.weight]

            if possible_vehicles:
                chosen_vehicle = random.choice(possible_vehicles)
                chosen_vehicle.packages.append(package)

        solution.fitness = self.evaluate_fitness(solution)
        return solution

    def create_ga_random_solution(self) -> Solution:
        solution = Solution(self.vehicles)
        packages_copy = self.packages.copy()
        random.shuffle(packages_copy)
        max_capacity = max(v.capacity for v in solution.vehicles)

        for package in packages_copy:
            if package.weight > max_capacity:
                continue

            valid_vehicles = [v for v in solution.vehicles
                              if v.available_capacity() >= package.weight]

            if valid_vehicles:
                chosen_vehicle = random.choice(valid_vehicles)
                chosen_vehicle.packages.append(package)

        solution.fitness = self.evaluate_fitness(solution)
        return solution

    def get_neighbor_solution(self, solution: Solution) -> Solution:
        neighbor = solution.deep_copy()
        move_type = random.choices([0, 1, 2, 3], weights=[0.4, 0.3, 0.2, 0.1], k=1)[0]

        if move_type == 0:  # Package move
            vehicles_with_packages = [v for v in neighbor.vehicles if v.packages]
            if vehicles_with_packages:
                source_vehicle = max(vehicles_with_packages, key=lambda v: v.utilization_ratio())
                package_to_move = max(source_vehicle.packages, key=lambda p: p.weight * (6 - p.priority))
                potential_targets = [v for v in neighbor.vehicles
                                     if v != source_vehicle and
                                     v.available_capacity() >= package_to_move.weight]

                if potential_targets:
                    target_vehicle = min(potential_targets,
                                         key=lambda v: (v.utilization_ratio(), -v.available_capacity()))
                    source_vehicle.packages.remove(package_to_move)
                    target_vehicle.packages.append(package_to_move)

        elif move_type == 1:
            vehicles = [v for v in neighbor.vehicles if v.packages]
            if len(vehicles) >= 2:
                vehicle1, vehicle2 = random.sample(vehicles, 2)
                if vehicle1.packages and vehicle2.packages:
                    p1 = random.choice(vehicle1.packages)
                    p2 = random.choice(vehicle2.packages)
                    if (vehicle1.available_capacity() + p1.weight >= p2.weight and
                            vehicle2.available_capacity() + p2.weight >= p1.weight):
                        vehicle1.packages.remove(p1)
                        vehicle2.packages.remove(p2)
                        vehicle1.packages.append(p2)
                        vehicle2.packages.append(p1)

        elif move_type == 2:  # Route optimization
            vehicle = random.choice([v for v in neighbor.vehicles if len(v.packages) >= 2])
            if len(vehicle.packages) >= 2:
                start = random.randint(0, len(vehicle.packages) - 2)
                end = random.randint(start + 1, len(vehicle.packages) - 1)
                vehicle.packages[start:end + 1] = reversed(vehicle.packages[start:end + 1])

        neighbor.fitness = self.evaluate_fitness(neighbor)
        return neighbor

    def simulated_annealing(self) -> Solution:
        initial_temp = 1000.0
        cooling_rate = 0.95
        stopping_temp = 1.0
        iterations_per_temp = 100

        current_solution = self.create_initial_solution()
        best_solution = current_solution.deep_copy()
        temperature = initial_temp

        while temperature > stopping_temp:
            for _ in range(iterations_per_temp):
                next_solution = self.get_neighbor_solution(current_solution)
                delta_fitness = next_solution.fitness - current_solution.fitness

                if delta_fitness < 0 or random.random() < math.exp(-delta_fitness / temperature):
                    current_solution = next_solution
                    if current_solution.fitness < best_solution.fitness:
                        best_solution = current_solution.deep_copy()

            temperature *= cooling_rate

        return best_solution

    def tournament_selection(self, population: List[Solution], tournament_size: int) -> Solution:
        tournament = random.sample(population, tournament_size)
        return min(tournament, key=lambda x: x.fitness)

    def crossover(self, parent1: Solution, parent2: Solution) -> Solution:
        child = Solution(self.vehicles)
        max_capacity = max(v.capacity for v in self.vehicles)

        for package in self.packages:
            if package.weight > max_capacity:
                continue

            parent_choice = random.choice([parent1, parent2])
            for i, vehicle in enumerate(parent_choice.vehicles):
                if package in vehicle.packages and child.vehicles[i].available_capacity() >= package.weight:
                    child.vehicles[i].packages.append(package)
                    break
            else:
                valid_vehicles = [v for v in child.vehicles if v.available_capacity() >= package.weight]
                if valid_vehicles:
                    chosen_vehicle = min(valid_vehicles, key=lambda v: v.utilization_ratio())
                    chosen_vehicle.packages.append(package)

        return child

    def mutate(self, solution: Solution) -> None:
        vehicles_with_packages = [v for v in solution.vehicles if v.packages]
        if not vehicles_with_packages:
            return

        source_vehicle = random.choice(vehicles_with_packages)
        if not source_vehicle.packages:
            return

        package_to_move = random.choice(source_vehicle.packages)
        potential_targets = [v for v in solution.vehicles
                             if v != source_vehicle and
                             v.available_capacity() >= package_to_move.weight]

        if potential_targets:
            target_vehicle = random.choice(potential_targets)
            source_vehicle.packages.remove(package_to_move)
            target_vehicle.packages.append(package_to_move)

    def genetic_algorithm(self) -> Solution:
        population_size = 100
        mutation_rate = 0.2
        max_generations = 200
        elite_size = 5
        tournament_size = 5

        population = [self.create_initial_solution()]
        for _ in range(population_size - 1):
            population.append(self.create_ga_random_solution())

        best_solution = min(population, key=lambda x: x.fitness).deep_copy()

        for generation in range(max_generations):
            population.sort(key=lambda x: x.fitness)

            if population[0].fitness < best_solution.fitness:
                best_solution = population[0].deep_copy()

            new_population = population[:elite_size]

            while len(new_population) < population_size:
                parent1 = self.tournament_selection(population, tournament_size)
                parent2 = self.tournament_selection(population, tournament_size)

                child = self.crossover(parent1, parent2)

                if random.random() < mutation_rate:
                    self.mutate(child)

                child.fitness = self.evaluate_fitness(child)
                new_population.append(child)

            population = new_population

        return best_solution

    def load_vehicles(self, file_path):
        self.vehicles = []
        with open(file_path, 'r') as file:
            id_counter = 1
            for line in file:
                line = line.strip()
                if line and not line.startswith('#'):
                    try:
                        capacity = float(line)
                        if capacity <= 0:
                            capacity = 100.0
                        self.vehicles.append(Vehicle(id_counter, capacity))
                        id_counter += 1
                    except ValueError:
                        continue

    def load_packages(self, file_path):
        self.packages = []
        with open(file_path, 'r') as file:
            id_counter = 1
            for line in file:
                line = line.strip()
                if line and not line.startswith('#'):
                    try:
                        if ',' in line:
                            values = line.split(',')
                        else:
                            values = line.split()

                        x = float(values[0])
                        y = float(values[1])
                        weight = float(values[2])
                        priority = int(values[3])

                        self.packages.append(Package(id_counter, x, y, weight, priority))
                        id_counter += 1
                    except (ValueError, IndexError):
                        continue

    def load_input_files(self):
        self.print_header()
        print("LOAD INPUT FILES")
        print("-" * 40)

        vehicles_file = input("Enter path to vehicles file (.txt): ")
        packages_file = input("Enter path to packages file (.txt): ")

        try:
            self.load_vehicles(vehicles_file)
            self.load_packages(packages_file)
            print(f"\nLoaded {len(self.vehicles)} vehicles and {len(self.packages)} packages!")
            input("Press Enter to continue...")
        except Exception as e:
            input(f"Error loading files: {str(e)}. Press Enter to continue...")

    def select_algorithm(self):
        self.print_header()
        print("SELECT OPTIMIZATION ALGORITHM")
        print("-" * 40)
        print("1. Simulated Annealing")
        print("2. Genetic Algorithm")

        choice = input("\nSelect algorithm (1-2): ")

        if choice == '1':
            self.current_algorithm = "simulated_annealing"
        elif choice == '2':
            self.current_algorithm = "genetic_algorithm"

        input("Press Enter to continue...")

    def run_optimization(self):
        self.print_header()
        print("RUNNING OPTIMIZATION")
        print("-" * 40)

        if not self.vehicles or not self.packages:
            input("Missing data. Press Enter to continue...")
            return

        max_capacity = max(v.capacity for v in self.vehicles)
        impossible_packages = [p for p in self.packages if p.weight > max_capacity]

        if impossible_packages:
            print(f"\n{len(impossible_packages)} packages exceed maximum capacity")
            input("Press Enter to continue...")
            return

        print(f"Using algorithm: {self.current_algorithm.replace('_', ' ').title()}")

        try:
            start_time = time.time()  # Record start time
            solution = self.simulated_annealing() if self.current_algorithm == "simulated_annealing" else self.genetic_algorithm()
            self.vehicles = copy.deepcopy(solution.vehicles)

            end_time = time.time()  # Record end time
            execution_time = end_time - start_time

            print(f"Execution time: {execution_time:.2f} seconds")

            self.display_results()
            self.display_visualization(solution)
        except Exception:
            print("\nOptimization failed - try different parameters")

        input("\nPress Enter to continue...")

    def display_results(self):
        self.print_header()
        print("DELIVERY OPTIMIZATION RESULTS")
        print("-" * 40)

        assigned_ids = {p.id for v in self.vehicles for p in v.packages}
        max_capacity = max(v.capacity for v in self.vehicles)
        total_capacity = sum(v.capacity for v in self.vehicles)
        total_weight = sum(p.weight for p in self.packages if p.weight <= max_capacity)

        unassignable_packages = [p for p in self.packages if p.weight > max_capacity]
        unassigned_but_possible = [p for p in self.packages
                                   if p.id not in assigned_ids and p.weight <= max_capacity]

        if unassignable_packages:
            print("\n🚫 Impossible Assignments:")
            print(f"  {len(unassignable_packages)} packages exceed maximum vehicle capacity ({max_capacity}kg)")
            print("  These packages cannot be delivered with current vehicles:")
            for p in sorted(unassignable_packages, key=lambda x: -x.weight):
                print(f"    - Package #{p.id}: {p.weight:.1f}kg (Priority {p.priority})")

        if unassigned_but_possible:
            print("\n⚠️ Capacity Limitations:")
            print(f"  {len(unassigned_but_possible)} packages couldn't be assigned due to insufficient capacity")
            print("  These packages remain undelivered:")
            for p in sorted(unassigned_but_possible, key=lambda x: (x.priority, -x.weight)):
                print(f"    - Package #{p.id}: {p.weight:.1f}kg (Priority {p.priority})")

        total_distance = 0.0
        active_vehicles = [v for v in self.vehicles if v.packages]
        overloaded_vehicles = [v for v in active_vehicles if v.is_overloaded()]

        if active_vehicles:
            print("\n📦 Delivery Assignments:")
            print(
                f"  {len(assigned_ids)}/{len([p for p in self.packages if p.weight <= max_capacity])} deliverable packages assigned")

            if overloaded_vehicles:
                print("\n🚨 Warning: Some vehicles are overloaded!")
                print("  Delivery may not be possible until capacity is adjusted")

            for vehicle in active_vehicles:
                distance = self.calculate_total_distance(vehicle)
                total_distance += distance

                status = "🚨 OVERLOADED" if vehicle.is_overloaded() else "✓ Within capacity"
                print(f"\nVehicle #{vehicle.id} ({status})")
                print(f"  Capacity: {vehicle.capacity:.1f}kg | Used: {vehicle.total_weight():.1f}kg")
                print(f"  Packages: {len(vehicle.packages)} | Route distance: {distance:.2f}km")

                print("  Delivery sequence (by priority):")
                current = self.shop_location
                for i, pkg in enumerate(sorted(vehicle.packages, key=lambda x: x.priority), 1):
                    dist = self.calculate_distance(current, (pkg.x, pkg.y))
                    print(f"    {i}. Package #{pkg.id} ({dist:.2f}km | {pkg.weight:.1f}kg | Priority {pkg.priority})")
                    current = (pkg.x, pkg.y)

            print(f"\nTotal delivery distance: {total_distance:.2f}km")
            print(
                f"Total capacity utilization: {sum(v.total_weight() for v in active_vehicles):.1f}/{total_capacity:.1f}kg")

            if not overloaded_vehicles and unassigned_but_possible:
                print("\nNote: Some packages remain unassigned due to capacity limitations")
                print("      Consider adding more vehicles or reducing package weights")
        else:
            print("\n❌ No packages could be assigned to vehicles")
            print("  All packages exceed vehicle capacities or no vehicles available")

    def display_visualization(self, solution: Solution):
        try:
            gui = DeliveryGUI(self)
            gui.update_visualization(solution)
            start_time = time.time()
            while gui.window_open and time.time() - start_time < 1000:
                gui.root.update()
                time.sleep(0.05)
            if gui.window_open:
                gui.close_window()
        except Exception as e:
            print(f"Visualization error: {str(e)}")

    def view_setup(self):
        self.print_header()
        print("CURRENT SETUP")
        print("-" * 40)
        print(f"Algorithm: {self.current_algorithm.replace('_', ' ').title()}")
        print(f"\nVehicles: {len(self.vehicles)}")
        print(f"Packages: {len(self.packages)}")
        input("\nPress Enter to continue...")

    def run_menu(self):
        while True:
            self.print_header()
            print("MAIN MENU")
            print("1. Load Input Files")
            print("2. Select Algorithm")
            print("3. Run Optimization")
            print("4. View Current Setup")
            print("5. Exit")

            choice = input("\nEnter choice (1-5): ")

            if choice == '1':
                self.load_input_files()
            elif choice == '2':
                self.select_algorithm()
            elif choice == '3':
                self.run_optimization()
            elif choice == '4':
                self.view_setup()
            elif choice == '5':
                print("\nExiting...")
                break


class DeliveryGUI:
    def __init__(self, delivery_system):
        self.delivery_system = delivery_system
        self.root = tk.Tk()
        self.root.title("Delivery Route Optimization")
        self.root.geometry("950x850")
        self.root.protocol("WM_DELETE_WINDOW", self.close_window)

        self.fig, self.ax = plt.subplots(figsize=(9.5, 8.5))
        self.canvas = FigureCanvasTkAgg(self.fig, master=self.root)
        self.canvas.get_tk_widget().pack(fill=tk.BOTH, expand=True)
        self.window_open = True

    def update_visualization(self, solution: Solution):
        self.ax.clear()

        self.ax.set_xlim(-10, 110)
        self.ax.set_ylim(-10, 110)
        self.ax.set_title("Optimized Delivery Routes", fontsize=12, pad=15)
        self.ax.set_xlabel("X Coordinate (km)", fontsize=9)
        self.ax.set_ylabel("Y Coordinate (km)", fontsize=9)
        self.ax.grid(True, linestyle=':', alpha=0.2)

        self.ax.plot(0, 0, 's', markersize=10, color='black', label='Shop')

        colors = ['#1f77b4', '#ff7f0e', '#2ca02c', '#d62728', '#9467bd']
        vehicle_lines = []

        for i, vehicle in enumerate(solution.vehicles):
            if not vehicle.packages:
                continue

            route = [(0, 0)] + [(p.x, p.y) for p in vehicle.packages] + [(0, 0)]
            x_coords, y_coords = zip(*route)

            line = self.ax.plot(x_coords, y_coords, '-',
                                color=colors[i % len(colors)],
                                linewidth=1.5,
                                alpha=0.8)[0]
            vehicle_lines.append(line)

            for j in range(len(x_coords) - 1):
                self.ax.annotate('',
                                 xy=(x_coords[j + 1], y_coords[j + 1]),
                                 xytext=(x_coords[j], y_coords[j]),
                                 arrowprops=dict(arrowstyle='->',
                                                 color=colors[i % len(colors)],
                                                 lw=1,
                                                 shrinkA=5,
                                                 shrinkB=5))

            for seq, package in enumerate(vehicle.packages, 1):
                priority_colors = {1: 'red', 2: 'orange', 3: 'gold', 4: 'silver', 5: 'gray'}
                self.ax.plot(package.x, package.y, 'o',
                             markersize=4 + package.weight / 8,
                             color=colors[i % len(colors)],
                             markeredgecolor=priority_colors.get(package.priority, 'black'),
                             markeredgewidth=1.5)

                self.ax.text(package.x, package.y + 3.5,
                             f"{seq}",
                             ha='center', va='center',
                             fontsize=8,
                             weight='bold',
                             bbox=dict(facecolor='white', alpha=0.8, pad=1, edgecolor='none'))

                self.ax.text(package.x, package.y - 3.5,
                             f"Pkg{package.id} (P{package.priority})",
                             ha='center', va='center',
                             fontsize=6,
                             bbox=dict(facecolor='white', alpha=0.7, pad=1, edgecolor='none'))

        stats_text = (f"Active Vehicles: {len([v for v in solution.vehicles if v.packages])}\n"
                      f"Total Packages: {sum(len(v.packages) for v in solution.vehicles)}")
        self.ax.text(1.02, 0.98, stats_text,
                     transform=self.ax.transAxes,
                     fontsize=8,
                     bbox=dict(facecolor='white', alpha=0.7))

        priority_legend = [
            plt.Line2D([0], [0], marker='o', color='w', label='Priority 1',
                       markerfacecolor='white', markeredgecolor='red', markersize=8),
            plt.Line2D([0], [0], marker='o', color='w', label='Priority 2',
                       markerfacecolor='white', markeredgecolor='orange', markersize=8),
            plt.Line2D([0], [0], marker='o', color='w', label='Priority 3',
                       markerfacecolor='white', markeredgecolor='gold', markersize=8),
            plt.Line2D([0], [0], marker='o', color='w', label='Priority 4-5',
                       markerfacecolor='white', markeredgecolor='gray', markersize=8)
        ]

        vehicle_legend = [
                             plt.Line2D([0], [0], marker='s', color='black', label='Shop',
                                        markersize=8, linestyle='None')
                         ] + [
                             plt.Line2D([0], [0], marker='o', color=colors[i % len(colors)],
                                        label=f'V{i + 1} ({v.capacity}kg)',
                                        markersize=8)
                             for i, v in enumerate(solution.vehicles) if v.packages
                         ]

        first_legend = self.ax.legend(handles=vehicle_legend,
                                      bbox_to_anchor=(1.02, 0.8),
                                      loc='upper left',
                                      fontsize=7,
                                      title="Vehicles",
                                      title_fontsize=8)

        self.ax.add_artist(first_legend)
        self.ax.legend(handles=priority_legend,
                       bbox_to_anchor=(1.02, 0.5),
                       loc='upper left',
                       fontsize=7,
                       title="Priorities",
                       title_fontsize=8)

        self.fig.tight_layout()
        self.canvas.draw()

    def close_window(self):
        plt.close(self.fig)
        self.root.destroy()
        self.window_open = False

if __name__ == "__main__":
    try:
        delivery_system = DeliverySystem()
        delivery_system.run_menu()
    except KeyboardInterrupt:
        print("\nProgram terminated by user.")
    except Exception as e:
        print(f"\nError: {str(e)}")
