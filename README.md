#include <climits>
#include <cstdint>
#include <iomanip>
#include <iostream>
#include <limits>
#include <stdexcept>
#include <string>

#include "bits.h"

#ifdef _WIN32
#include <windows.h>
#endif

using bits::BitOp;

namespace {

void setupConsole() {
#ifdef _WIN32
    SetConsoleOutputCP(CP_UTF8);  // исходники сохранять в UTF-8
    SetConsoleCP(CP_UTF8);
#endif
}

void section(const std::string& title) { std::cout << "\n=== " << title << " ===\n"; }

const char* opName(BitOp op) {
    switch (op) {
        case BitOp::Clear:  return "сброс (&= ~mask)";
        case BitOp::Set:    return "установка (|= mask)";
        default:            return "инверсия (^= mask)";
    }
}

// --- вывод значений ---
void showInt(std::int32_t v) {
    std::cout << std::setw(12) << v << " : " << bits::intBits(v) << "\n";
}
void showUInt(std::uint32_t v) {
    std::cout << std::setw(12) << v << " : " << bits::uintBits(v) << "\n";
}
void showFloat(float v) {
    std::cout << std::setw(12) << v << " : " << bits::floatBits(v) << "  [" << bits::floatKind(v) << "]\n";
}
void showDouble(double v) {
    std::cout << std::setw(12) << v << " : " << bits::doubleBits(v) << "  [" << bits::doubleKind(v) << "]\n";
}

std::string bitsOf(std::int32_t v) { return bits::intBits(v); }
std::string bitsOf(float v)        { return bits::floatBits(v); }
std::string bitsOf(double v)       { return bits::doubleBits(v); }

template <class T>
void showChange(T value, int pos, BitOp op) {
    const T result = bits::changeBit(value, pos, op);
    std::cout << opName(op) << ", бит " << pos << ":\n"
              << "  до:    " << bitsOf(value) << "  = " << value << "\n"
              << "  после: " << bitsOf(result) << "  = " << result << "\n";
}

void showMul(std::int32_t v) {
    bool overflow = false;
    const std::int32_t r = bits::mulBy2(v, overflow);
    std::cout << "  " << v << " << 1\n"
              << "    до:    " << bits::intBits(v) << "\n"
              << "    после: " << bits::intBits(r) << "  = " << r
              << (overflow ? "   <- переполнение, результат != " + std::to_string(v) + " * 2" : "")
              << "\n";
}

// --- автоматическая демонстрация (требование 6) ---
void demo() {
    const float  fInf  = std::numeric_limits<float>::infinity();
    const double dInf  = std::numeric_limits<double>::infinity();
    const double dDen  = std::numeric_limits<double>::denorm_min();

    section("Задание 1. Размеры типов");
    bits::printTypeSizes();

    section("Задание 2. Целые числа (знак | значащие разряды)");
    for (std::int32_t v : {42, -42, 0, INT_MAX, INT_MIN}) showInt(v);
    std::cout << "unsigned (знакового разряда нет):\n";
    showUInt(42u);
    showUInt(4000000000u);
    std::cout << "Коды числа 42:\n";  bits::printIntCodes(42);
    std::cout << "Коды числа -42:\n"; bits::printIntCodes(-42);

    section("Задание 3. float (знак | порядок | мантисса)");
    showFloat(3.14f);
    showFloat(-2.5f);
    showFloat(0.0f);
    showFloat(1e-40f);   // денормализованное
    showFloat(fInf);
    showFloat(std::numeric_limits<float>::quiet_NaN());

    section("Задание 4. double (знак | порядок | мантисса)");
    showDouble(3.14);
    showDouble(-2.5);
    showDouble(0.0);
    showDouble(dDen);    // наименьшее денормализованное
    showDouble(dInf);

    section("Задание 5. Изменение бита");
    showChange<std::int32_t>(42, 0, BitOp::Set);
    showChange<std::int32_t>(42, 1, BitOp::Clear);
    showChange<std::int32_t>(42, 31, BitOp::Toggle);   // смена знака
    showChange<std::int32_t>(-1, 31, BitOp::Clear);
    showChange(3.14f, 31, BitOp::Toggle);               // float: смена знака
    showChange(1e-40f, 23, BitOp::Set);                 // float: денормализованное -> нормализованное
    showChange(-2.5, 63, BitOp::Clear);                 // double: сброс знака
    showChange(1.0, 62, BitOp::Toggle);                 // double: меняется порядок

    section("ИДЗ. Умножение на 2 через сдвиг");
    for (std::int32_t v : {21, -7, 0, 1 << 30, INT_MAX, INT_MIN}) showMul(v);
}

template <class T>
bool readValue(const char* prompt, T& out) {
    std::cout << prompt;
    if (std::cin >> out) return true;
    if (std::cin.eof()) return false;
    std::cin.clear();
    std::cin.ignore(std::numeric_limits<std::streamsize>::max(), '\n');
    std::cout << "Ошибка ввода, нужно число.\n";
    return false;
}

template <class T>
void interactiveChange(const char* typeName, int maxBit) {
    T value{};
    int pos = 0, op = 0;
    if (!readValue((std::string("Введите ") + typeName + ": ").c_str(), value)) return;
    std::cout << "Номер бита (0.." << maxBit << ", 0 - младший): ";
    if (!readValue("", pos)) return;
    if (!readValue("Операция (1 - сбросить, 2 - установить, 3 - инвертировать): ", op)) return;
    if (op < 1 || op > 3) { std::cout << "Ошибка. Операция 1-3.\n"; return; }
    try {
        showChange(value, pos, static_cast<BitOp>(op));
    } catch (const std::out_of_range& e) {
        std::cout << "Ошибка: " << e.what() << "\n";
    }
}

void interactive() {
    for (;;) {
        std::cout << "\n--- Интерактивный режим ---\n"
                  << "1 - int   2 - float   3 - double   4 - изменить бит   5 - умножить на 2 (ИДЗ)   0 - выход\n";
        int cmd = -1;
        if (!readValue("Выбор: ", cmd)) { if (std::cin.eof()) return; continue; }

        if (cmd == 0) return;
        if (cmd == 1) { std::int32_t v; if (readValue("Целое число: ", v)) { showInt(v); bits::printIntCodes(v); } }
        else if (cmd == 2) { float v;  if (readValue("float: ", v))  showFloat(v); }
        else if (cmd == 3) { double v; if (readValue("double: ", v)) showDouble(v); }
        else if (cmd == 4) {
            int type = 0;
            if (!readValue("Тип (1 - int, 2 - float, 3 - double): ", type)) continue;
            if (type == 1)      interactiveChange<std::int32_t>("int", 31);
            else if (type == 2) interactiveChange<float>("float", 31);
            else if (type == 3) interactiveChange<double>("double", 63);
            else std::cout << "Ошибка. Неверный тип.\n";
        }
        else if (cmd == 5) { std::int32_t v; if (readValue("Целое число: ", v)) showMul(v); }
        else std::cout << "Неизвестная команда.\n";
    }
}

}  // namespace

int main() {
    setupConsole();
    demo();

    int answer = 0;
    std::cout << "\nПерейти в интерактивный режим? (1 - да, 0 - нет): ";
    if (std::cin >> answer && answer == 1) interactive();
    return 0;
}
