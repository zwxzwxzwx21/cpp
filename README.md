#include <iostream>
#include <vector>
#include <string>
#include <fstream>
#include <algorithm> // Dla std::sort, std::for_each, std::find_if
#include <stdexcept> // Dla wyjątków
#include <memory>    // Dla std::unique_ptr
#include <iomanip>   // Dla std::fixed, std::setprecision
#include <limits>    // Dla std::numeric_limits

// --- Lab 7: Klasy Abstrakcyjne i Składowe Statyczne ---

// Deklaracja wyprzedzająca dla funkcji zaprzyjaźnionej operator<<
class Product;
std::ostream& operator<<(std::ostream& os, const Product& p);

// Klasa bazowa abstrakcyjna
class Product {
private:
    static int productCount; // Statyczne pole - licznik wszystkich produktów
protected:
    std::string name;
    double price; // Cena przechowywana w klasie bazowej dla uproszczenia
public:
    Product(std::string n, double pr) : name(std::move(n)), price(pr) {
        if (pr < 0) {
            // --- Lab 10: Wyjątki ---
            throw std::invalid_argument("Cena produktu nie może być ujemna: " + name);
        }
        productCount++;
    }

    // Wirtualny destruktor jest kluczowy dla poprawnego zwalniania pamięci
    // przy użyciu wskaźników do klasy bazowej.
    virtual ~Product() {
        productCount--;
    }

    // Metody czysto wirtualne czynią Product klasą abstrakcyjną
    virtual std::string getType() const = 0; // Np. "Book", "Electronics"
    virtual std::string getSpecificDescription() const = 0; // Szczegółowy opis dla danego typu

    // Metoda wirtualna z domyślną implementacją, może być nadpisana
    virtual void displayFullDetails() const {
        std::cout << "Typ: " << getType() << "\n"
                  << "Nazwa: " << name << "\n"
                  << "Cena: $" << std::fixed << std::setprecision(2) << price << "\n"
                  << "Opis: " << getSpecificDescription() << std::endl;
    }

    std::string getName() const { return name; }
    double getPrice() const { return price; }

    // Metoda statyczna
    static int getProductCount() {
        return productCount;
    }

    // --- Lab 8/9: Przeciążanie Operatorów ---
    friend std::ostream& operator<<(std::ostream& os, const Product& p);

    // Operator do sortowania (np. po cenie)
    bool operator<(const Product& other) const {
        return this->price < other.price;
    }

    // Wirtualna metoda do zapisu specyficznych danych do pliku
    virtual void saveDataToFile(std::ostream& os) const {
        os << name << std::endl;
        os << price << std::endl;
        os << getSpecificDescription() << std::endl;
    }
};

// Definicja i inicjalizacja statycznego pola
int Product::productCount = 0;

// Implementacja przeciążonego operatora<< dla Product
std::ostream& operator<<(std::ostream& os, const Product& p) {
    os << "[" << p.getType() << "] " << p.getName()
       << " - $" << std::fixed << std::setprecision(2) << p.getPrice()
       << " (Opis: " << p.getSpecificDescription() << ")";
    return os;
}

// Klasy pochodne
class Book : public Product {
private:
    std::string author;
    int pages;
public:
    Book(const std::string& n, double pr, std::string auth, int pg)
        : Product(n, pr), author(std::move(auth)), pages(pg) {}

    std::string getType() const override { return "Book"; }
    std::string getSpecificDescription() const override {
        return "Autor: " + author + ", Stron: " + std::to_string(pages);
    }

    // Nadpisanie metody do zapisu specyficznych danych
    void saveDataToFile(std::ostream& os) const override {
        Product::saveDataToFile(os); // Wywołaj metodę bazową, jeśli zapisuje wspólne dane
                                     // W tym przypadku Product::saveDataToFile już zapisuje name, price, desc
                                     // Jeśli chcemy bardziej szczegółowo:
        // os << name << std::endl;
        // os << price << std::endl;
        os << author << std::endl; // Zapisz specyficzne dla Book
        os << pages << std::endl;
    }
};

class Electronics : public Product {
private:
    std::string brand;
    int warrantyMonths;
public:
    Electronics(const std::string& n, double pr, std::string br, int warranty)
        : Product(n, pr), brand(std::move(br)), warrantyMonths(warranty) {}

    std::string getType() const override { return "Electronics"; }
    std::string getSpecificDescription() const override {
        return "Marka: " + brand + ", Gwarancja: " + std::to_string(warrantyMonths) + " mies.";
    }
    
    void saveDataToFile(std::ostream& os) const override {
        // Product::saveDataToFile(os); // Jak wyżej, zależy od struktury
        os << name << std::endl; // Powtórzenie dla spójności odczytu
        os << price << std::endl;
        os << brand << std::endl;
        os << warrantyMonths << std::endl;
    }
};

// --- Lab 11: Szablony ---
// Funkcja szablonowa do wyświetlania dowolnego elementu, który ma operator<<
template <typename T>
void printGenericItem(const std::string& label, const T& item) {
    std::cout << label << ": " << item << std::endl;
}

// Prosta klasa szablonowa - para klucz-wartość
template <typename K, typename V>
class Attribute {
public:
    K key;
    V value;
    Attribute(K k, V v) : key(std::move(k)), value(std::move(v)) {}

    friend std::ostream& operator<<(std::ostream& os, const Attribute<K,V>& attr) {
        os << attr.key << " -> " << attr.value;
        return os;
    }
};


// --- Lab 14: Zapis i Odczyt Plików ---
// Uproszczony zapis i odczyt, tekstowy. Wymaga znajomości formatu.
void saveInventory(const std::vector<std::unique_ptr<Product>>& inventory, const std::string& filename) {
    std::ofstream outFile(filename);
    if (!outFile) {
        throw std::runtime_error("Błąd: Nie można otworzyć pliku do zapisu: " + filename);
    }
    for (const auto& product_ptr : inventory) {
        if (product_ptr) {
            outFile << product_ptr->getType() << std::endl; // Zapisz typ do identyfikacji przy odczycie
            product_ptr->saveDataToFile(outFile); // Każdy produkt zapisuje swoje dane
        }
    }
    std::cout << "Inwentarz zapisany do pliku: " << filename << std::endl;
}

std::vector<std::unique_ptr<Product>> loadInventory(const std::string& filename) {
    std::vector<std::unique_ptr<Product>> inventory;
    std::ifstream inFile(filename);
    if (!inFile) {
        std::cerr << "Ostrzeżenie: Nie można otworzyć pliku do odczytu: " << filename << ". Start z pustym inwentarzem." << std::endl;
        return inventory;
    }

    std::string type, name, author_brand, desc_line;
    double price;
    int pages_warranty;

    while (std::getline(inFile, type)) {
        try {
            if (!std::getline(inFile, name)) break;
            if (!(inFile >> price)) break;
            inFile.ignore(std::numeric_limits<std::streamsize>::max(), '\n'); // Ignoruj resztę linii po cenie

            if (type == "Book") {
                if (!std::getline(inFile, author_brand)) break; // Autor
                if (!(inFile >> pages_warranty)) break;       // Strony
                inFile.ignore(std::numeric_limits<std::streamsize>::max(), '\n');
                inventory.push_back(std::make_unique<Book>(name, price, author_brand, pages_warranty));
            } else if (type == "Electronics") {
                if (!std::getline(inFile, author_brand)) break; // Marka
                if (!(inFile >> pages_warranty)) break;       // Gwarancja
                inFile.ignore(std::numeric_limits<std::streamsize>::max(), '\n');
                inventory.push_back(std::make_unique<Electronics>(name, price, author_brand, pages_warranty));
            } else {
                std::cerr << "Nieznany typ produktu w pliku: " << type << ". Pomijanie." << std::endl;
                // Przeskocz linie, które miałyby być danymi tego nieznanego produktu
                // To jest uproszczenie, w realnym systemie trzeba by wiedzieć ile linii pominąć
                // lub mieć bardziej znacznikowany format.
                // Dla tego formatu, jeśli Product::saveDataToFile zapisuje 3 linie, a pochodne 2 dodatkowe,
                // to po name i price, jeszcze 1 linia dla Product::getSpecificDescription()
                // a potem specyficzne. Nasz saveDataToFile w pochodnych zapisuje wszystko.
                // Dla Book: name, price, author, pages (4 linie po typie)
                // Dla Electronics: name, price, brand, warranty (4 linie po typie)
                // Jeśli Product::saveDataToFile zapisywałby name, price, desc, to 3 linie.
                // Nasz obecny format: typ, name, price, spec1, spec2
                // Więc po odczytaniu name i price, jeszcze 2 linie do pominięcia jeśli typ nieznany.
                if (std::getline(inFile, desc_line) && std::getline(inFile, desc_line)) {
                    // Pomyślnie pominięto 2 linie
                }
            }
        } catch (const std::invalid_argument& e) {
            std::cerr << "Błąd podczas tworzenia produktu z pliku (dane: " << name << "): " << e.what() << std::endl;
        } catch (const std::exception& e) {
            std::cerr << "Ogólny błąd parsowania produktu " << name << ": " << e.what() << std::endl;
        }
    }
    std::cout << "Inwentarz załadowany z pliku: " << filename << std::endl;
    return inventory;
}


int main() {
    std::cout << "--- Zaawansowany Przykład C++ ---" << std::endl;
    std::cout << "Początkowa liczba produktów (statyczna): " << Product::getProductCount() << std::endl;

    // --- Lab 12/13: STL Kontenery (std::vector z std::unique_ptr) ---
    std::vector<std::unique_ptr<Product>> inventory;
    const std::string inventoryFilename = "inventory_data.txt";

    // Załaduj inwentarz z pliku
    inventory = loadInventory(inventoryFilename);
    std::cout << "Liczba produktów po załadowaniu: " << Product::getProductCount() << std::endl;

    // Dodawanie produktów z obsługą wyjątków
    std::cout << "\n--- Dodawanie produktów ---" << std::endl;
    try {
        inventory.push_back(std::make_unique<Book>("Władca Pierścieni", 25.99, "J.R.R. Tolkien", 1200));
        inventory.push_back(std::make_unique<Electronics>("Smartfon Galaxy S25", 999.99, "Samsung", 24));
        inventory.push_back(std::make_unique<Book>("Czysty Kod", 35.50, "Robert C. Martin", 464));
        // Próba dodania produktu z niepoprawną ceną (powinien rzucić wyjątek)
        // inventory.push_back(std::make_unique<Electronics>("Wadliwy TV", -100.0, "BadBrand", 6));
    } catch (const std::invalid_argument& e) {
        std::cerr << "BŁĄD DODAWANIA: " << e.what() << std::endl;
    } catch (const std::exception& e) { // Ogólny handler
        std::cerr << "NIESPODZIEWANY BŁĄD: " << e.what() << std::endl;
    }

    std::cout << "Liczba produktów po dodaniu: " << Product::getProductCount() << std::endl;

    // Wyświetlanie inwentarza (polimorfizm, operator<<)
    std::cout << "\n--- Aktualny Inwentarz (niesortowany) ---" << std::endl;
    // --- Lab 12/13: STL Algorytmy (std::for_each) i Iteratory (niejawnie) ---
    std::for_each(inventory.begin(), inventory.end(), [](const auto& product_ptr){
        if (product_ptr) printGenericItem("Produkt", *product_ptr); // Użycie funkcji szablonowej
    });

    // Sortowanie inwentarza (używa Product::operator<)
    std::cout << "\n--- Sortowanie Inwentarza (po cenie) ---" << std::endl;
    std::sort(inventory.begin(), inventory.end(), [](const auto& a, const auto& b){
        // Potrzebne dla std::unique_ptr, aby porównywać obiekty, na które wskazują
        if (!a) return true; // nulls na początek lub koniec, zależy od logiki
        if (!b) return false;
        return *a < *b;
    });

    std::cout << "\n--- Inwentarz Posortowany ---" << std::endl;
    for(const auto& product_ptr : inventory) {
        if (product_ptr) {
            product_ptr->displayFullDetails(); // Polimorficzne wywołanie metody wirtualnej
            std::cout << "--------------------" << std::endl;
        }
    }
    
    // Użycie klasy szablonowej
    Attribute<std::string, int> warrantyDetails("Standardowa Gwarancja", 12);
    printGenericItem("\nSzczegół Atrybutu", warrantyDetails);

    // Zapis inwentarza do pliku
    try {
        saveInventory(inventory, inventoryFilename);
    } catch (const std::runtime_error& e) {
        std::cerr << "BŁĄD ZAPISU: " << e.what() << std::endl;
    }

    std::cout << "\nKońcowa liczba produktów (przed zwolnieniem pamięci): " << Product::getProductCount() << std::endl;
    // std::unique_ptr automatycznie zwolni pamięć, wywołując destruktory Product,
    // co zaktualizuje statyczny licznik productCount.

    return 0;
}
