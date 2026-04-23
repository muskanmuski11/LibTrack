"""
Library Book Tracker – Merged Version
Created by: Muskan Murmu
Merged from: muskan.py, tracker.py, db.py, cli.py, __init__.py, test_muskan.py

Features:
  - SQLite backend (no MySQL setup required; switchable via .env)
  - Add, View, Modify, Delete, Search books
  - Borrow / Return with transaction log
  - Low-stock report & category-wise count
  - Password-protected menu
  - Full unit-test suite (run with: python library_tracker.py --test)
"""

import os
import sys
import sqlite3
import unittest
from datetime import datetime
from unittest.mock import patch, MagicMock

# ─────────────────────────────────────────────
# Optional: load .env for DB path / password
# ─────────────────────────────────────────────
try:
    from dotenv import load_dotenv
    load_dotenv()
except ImportError:
    pass  # python-dotenv not installed – that's fine

# ─────────────────────────────────────────────
# Configuration  (override via .env)
# ─────────────────────────────────────────────
DB_PATH   = os.getenv("DB_PATH",     os.path.join(os.path.dirname(os.path.abspath(__file__)), "library.db"))
APP_PASSWORD = os.getenv("APP_PASSWORD", "readandlearn1111")
LOW_STOCK_THRESHOLD = int(os.getenv("LOW_STOCK_THRESHOLD", "2"))


# ══════════════════════════════════════════════
#  DATABASE LAYER  (from db.py + tracker.py)
# ══════════════════════════════════════════════

def get_connection() -> sqlite3.Connection | None:
    """Open and return a SQLite connection. Returns None on failure."""
    try:
        conn = sqlite3.connect(DB_PATH)
        conn.execute("PRAGMA foreign_keys = ON")
        return conn
    except sqlite3.Error as e:
        print(f"[DB] Unable to connect: {e}")
        return None


def init_db():
    """Create tables if they don't already exist."""
    conn = get_connection()
    if not conn:
        sys.exit(1)
    try:
        conn.executescript("""
            CREATE TABLE IF NOT EXISTS books (
                id        INTEGER PRIMARY KEY AUTOINCREMENT,
                title     TEXT    NOT NULL,
                author    TEXT    NOT NULL,
                isbn      TEXT    UNIQUE,
                category  TEXT    DEFAULT 'Uncategorized',
                quantity  INTEGER DEFAULT 1,
                available INTEGER DEFAULT 1
            );

            CREATE TABLE IF NOT EXISTS transactions (
                id       INTEGER PRIMARY KEY AUTOINCREMENT,
                book_id  INTEGER NOT NULL,
                action   TEXT    NOT NULL,   -- 'borrow' or 'return'
                date     TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                FOREIGN KEY(book_id) REFERENCES books(id)
            );
        """)
        conn.commit()
    except sqlite3.Error as e:
        print(f"[DB] Error initialising tables: {e}")
    finally:
        conn.close()


def execute_query(query: str, params: tuple = ()) -> bool:
    """Run a write query (INSERT / UPDATE / DELETE). Returns True on success."""
    conn = get_connection()
    if not conn:
        return False
    try:
        conn.execute(query, params)
        conn.commit()
        return True
    except sqlite3.IntegrityError as e:
        print(f"[DB] Integrity error: {e}")
    except sqlite3.Error as e:
        print(f"[DB] Query error: {e}")
    finally:
        conn.close()
    return False


def fetch_all(query: str, params: tuple = ()) -> list:
    """Run a SELECT and return all rows."""
    conn = get_connection()
    if not conn:
        return []
    try:
        cur = conn.execute(query, params)
        return cur.fetchall()
    except sqlite3.Error as e:
        print(f"[DB] Fetch error: {e}")
        return []
    finally:
        conn.close()


def fetch_one(query: str, params: tuple = ()):
    """Run a SELECT and return the first row (or None)."""
    conn = get_connection()
    if not conn:
        return None
    try:
        cur = conn.execute(query, params)
        return cur.fetchone()
    except sqlite3.Error as e:
        print(f"[DB] Fetch error: {e}")
        return None
    finally:
        conn.close()


# ══════════════════════════════════════════════
#  BOOK OPERATIONS  (from muskan.py + tracker.py)
# ══════════════════════════════════════════════

def add_book():
    """Prompt user and insert a new book."""
    print("\n--- Add a New Book ---")
    title    = input("Enter book title: ").strip()
    author   = input("Enter author: ").strip()
    isbn     = input("Enter ISBN (optional): ").strip() or None
    category = input("Enter category (default: Uncategorized): ").strip() or "Uncategorized"

    try:
        quantity = int(input("Enter quantity (default 1): ").strip() or "1")
    except ValueError:
        print("Quantity must be a number. Setting to 1.")
        quantity = 1

    ok = execute_query(
        "INSERT INTO books (title, author, isbn, category, quantity, available) VALUES (?, ?, ?, ?, ?, ?)",
        (title, author, isbn, category, quantity, quantity)
    )
    if ok:
        print(f"Book '{title}' added successfully!")


def view_all_books():
    """Display all books in a formatted table."""
    print("\n--- Library Catalog ---")
    rows = fetch_all("SELECT id, title, author, isbn, category, available, quantity FROM books ORDER BY title")

    if not rows:
        print("No books in the library yet.")
        return

    print(f"{'ID':<5} {'Title':<30} {'Author':<20} {'Category':<15} {'Avail/Total':<12} {'ISBN'}")
    print("─" * 95)
    for book_id, title, author, isbn, category, available, quantity in rows:
        print(
            f"{book_id:<5} {title[:29]:<30} {author[:19]:<20} {(category or '')[:14]:<15} "
            f"{available}/{quantity:<10} {isbn or 'N/A'}"
        )


def modify_book():
    """Update details of an existing book (look up by title)."""
    print("\n--- Modify Book Details ---")
    title = input("Enter the title of the book to modify: ").strip()

    book = fetch_one("SELECT * FROM books WHERE title = ?", (title,))
    if not book:
        print("Book not found.")
        return

    # book columns: id, title, author, isbn, category, quantity, available
    bid, old_title, old_author, old_isbn, old_cat, old_qty, old_avail = book

    print("Press Enter to keep the current value.")
    new_title    = input(f"Title [{old_title}]: ").strip()    or old_title
    new_author   = input(f"Author [{old_author}]: ").strip()  or old_author
    new_isbn     = input(f"ISBN [{old_isbn}]: ").strip()      or old_isbn
    new_category = input(f"Category [{old_cat}]: ").strip()   or old_cat

    new_qty_raw = input(f"Quantity [{old_qty}]: ").strip()
    try:
        new_qty = int(new_qty_raw) if new_qty_raw else old_qty
    except ValueError:
        print("Invalid quantity – keeping old value.")
        new_qty = old_qty

    # Keep available proportional if total qty changed
    qty_delta   = new_qty - old_qty
    new_avail   = max(0, old_avail + qty_delta)

    ok = execute_query(
        "UPDATE books SET title=?, author=?, isbn=?, category=?, quantity=?, available=? WHERE id=?",
        (new_title, new_author, new_isbn, new_category, new_qty, new_avail, bid)
    )
    if ok:
        print("Book updated successfully!")


def delete_book():
    """Delete a book by title after confirmation."""
    print("\n--- Delete a Book ---")
    title = input("Enter book title to delete: ").strip()

    book = fetch_one("SELECT id FROM books WHERE title = ?", (title,))
    if not book:
        print("Book not found.")
        return

    confirm = input(f"Are you sure you want to delete '{title}'? (y/n): ").lower()
    if confirm == "y":
        execute_query("DELETE FROM transactions WHERE book_id = ?", (book[0],))
        execute_query("DELETE FROM books WHERE id = ?", (book[0],))
        print("Book deleted successfully.")
    else:
        print("Deletion cancelled.")


def search_books():
    """Search by partial title or author."""
    print("\n--- Search Books ---")
    keyword = input("Enter title or author keyword: ").strip()
    pattern = f"%{keyword}%"

    rows = fetch_all(
        "SELECT id, title, author, category, available, quantity FROM books "
        "WHERE title LIKE ? OR author LIKE ?",
        (pattern, pattern)
    )
    if not rows:
        print("No matching books found.")
        return

    print(f"\nFound {len(rows)} match(es):")
    print(f"{'ID':<5} {'Title':<30} {'Author':<20} {'Category':<15} {'Avail/Total'}")
    print("─" * 80)
    for book_id, title, author, category, available, quantity in rows:
        print(f"{book_id:<5} {title[:29]:<30} {author[:19]:<20} {(category or '')[:14]:<15} {available}/{quantity}")


# ── Borrow / Return (from tracker.py) ──────────

def borrow_book():
    """Decrement available count and log a borrow transaction."""
    print("\n--- Borrow a Book ---")
    try:
        book_id = int(input("Enter Book ID to borrow: ").strip())
    except ValueError:
        print("Invalid input – please enter a numeric ID.")
        return

    row = fetch_one("SELECT title, available FROM books WHERE id = ?", (book_id,))
    if not row:
        print("Book not found.")
        return

    title, available = row
    if available > 0:
        conn = get_connection()
        if conn:
            try:
                conn.execute("UPDATE books SET available = available - 1 WHERE id = ?", (book_id,))
                conn.execute("INSERT INTO transactions (book_id, action) VALUES (?, 'borrow')", (book_id,))
                conn.commit()
                print(f"You have borrowed '{title}'. Enjoy reading!")
            except sqlite3.Error as e:
                print(f"Transaction error: {e}")
            finally:
                conn.close()
    else:
        print(f"Sorry, '{title}' is currently out of stock.")


def return_book():
    """Increment available count and log a return transaction."""
    print("\n--- Return a Book ---")
    try:
        book_id = int(input("Enter Book ID to return: ").strip())
    except ValueError:
        print("Invalid input – please enter a numeric ID.")
        return

    row = fetch_one("SELECT title, available, quantity FROM books WHERE id = ?", (book_id,))
    if not row:
        print("Book not found.")
        return

    title, available, quantity = row
    if available < quantity:
        conn = get_connection()
        if conn:
            try:
                conn.execute("UPDATE books SET available = available + 1 WHERE id = ?", (book_id,))
                conn.execute("INSERT INTO transactions (book_id, action) VALUES (?, 'return')", (book_id,))
                conn.commit()
                print(f"You have returned '{title}'. Thank you!")
            except sqlite3.Error as e:
                print(f"Transaction error: {e}")
            finally:
                conn.close()
    else:
        print(f"All copies of '{title}' are already in the library.")


# ── Reports (from muskan.py) ────────────────────

def low_stock_report():
    """List books with available copies ≤ LOW_STOCK_THRESHOLD."""
    print(f"\n--- Low Stock Report (threshold ≤ {LOW_STOCK_THRESHOLD}) ---")
    rows = fetch_all(
        "SELECT title, available, quantity FROM books WHERE available <= ?",
        (LOW_STOCK_THRESHOLD,)
    )
    if rows:
        print(f"{'Title':<35} {'Available':<10} {'Total'}")
        print("─" * 55)
        for title, available, quantity in rows:
            print(f"{title[:34]:<35} {available:<10} {quantity}")
    else:
        print("All books have sufficient stock.")


def category_report():
    """Show total and available copies grouped by category."""
    print("\n--- Books by Category ---")
    rows = fetch_all(
        "SELECT category, COUNT(*) AS titles, SUM(quantity) AS total, SUM(available) AS avail "
        "FROM books GROUP BY category ORDER BY category"
    )
    if rows:
        print(f"{'Category':<20} {'Titles':<8} {'Total Copies':<14} {'Available'}")
        print("─" * 55)
        for category, titles, total, avail in rows:
            print(f"{(category or 'Uncategorized')[:19]:<20} {titles:<8} {total:<14} {avail}")
    else:
        print("No data available.")


def transaction_log():
    """Show the most recent 20 borrow/return events."""
    print("\n--- Recent Transactions (last 20) ---")
    rows = fetch_all(
        "SELECT t.id, b.title, t.action, t.date "
        "FROM transactions t JOIN books b ON t.book_id = b.id "
        "ORDER BY t.date DESC LIMIT 20"
    )
    if rows:
        print(f"{'#':<5} {'Title':<30} {'Action':<8} {'Date'}")
        print("─" * 65)
        for tid, title, action, date in rows:
            print(f"{tid:<5} {title[:29]:<30} {action:<8} {date}")
    else:
        print("No transactions recorded yet.")


# ══════════════════════════════════════════════
#  UNIT TESTS  (from test_muskan.py, adapted)
# ══════════════════════════════════════════════

class TestLibraryTracker(unittest.TestCase):

    def setUp(self):
        """Use a file-based temp DB so all helpers share the same data."""
        import tempfile, library_tracker as lt
        self._tmp = tempfile.NamedTemporaryFile(suffix=".db", delete=False)
        self._tmp.close()
        lt.DB_PATH = self._tmp.name
        init_db()   # create tables in the temp file

    def tearDown(self):
        import library_tracker as lt
        try:
            os.unlink(lt.DB_PATH)
        except OSError:
            pass

    def _insert_book(self, title="Test Book", author="Test Author",
                     category="Fiction", quantity=5):
        execute_query(
            "INSERT INTO books (title, author, category, quantity, available) VALUES (?,?,?,?,?)",
            (title, author, category, quantity, quantity)
        )

    def _get_row(self, query, params=()):
        return fetch_one(query, params)

    # -- add_book -----------------------------------------------------------
    def test_add_book_inserts_row(self):
        with patch("builtins.input", side_effect=["My Book", "Some Author", "", "Fiction", "3"]):
            add_book()
        row = fetch_one("SELECT title, quantity FROM books WHERE title='My Book'")
        self.assertIsNotNone(row)
        self.assertEqual(row[1], 3)

    # -- view_all_books ------------------------------------------------------
    def test_view_all_books_no_crash(self):
        self._insert_book()
        with patch("builtins.print"):
            view_all_books()   # should not raise

    # -- search_books --------------------------------------------------------
    def test_search_finds_match(self):
        self._insert_book(title="Python Basics")
        with patch("builtins.input", return_value="Python"), \
             patch("builtins.print") as mock_print:
            search_books()
        output = " ".join(str(c) for c in mock_print.call_args_list)
        self.assertIn("Python", output)

    def test_search_no_match(self):
        with patch("builtins.input", return_value="ZZZNOMATCH"), \
             patch("builtins.print") as mock_print:
            search_books()
        output = " ".join(str(c) for c in mock_print.call_args_list)
        self.assertIn("No matching", output)

    # -- borrow_book / return_book -------------------------------------------
    def test_borrow_decrements_available(self):
        self._insert_book(quantity=3)
        book_id = fetch_one("SELECT id FROM books")[0]
        with patch("builtins.input", return_value=str(book_id)):
            borrow_book()
        avail = fetch_one("SELECT available FROM books WHERE id=?", (book_id,))[0]
        self.assertEqual(avail, 2)

    def test_return_increments_available(self):
        self._insert_book(quantity=3)
        book_id = fetch_one("SELECT id FROM books")[0]
        execute_query("UPDATE books SET available = available - 1 WHERE id = ?", (book_id,))
        with patch("builtins.input", return_value=str(book_id)):
            return_book()
        avail = fetch_one("SELECT available FROM books WHERE id=?", (book_id,))[0]
        self.assertEqual(avail, 3)

    def test_borrow_out_of_stock(self):
        self._insert_book(quantity=1)
        book_id = fetch_one("SELECT id FROM books")[0]
        execute_query("UPDATE books SET available = 0 WHERE id = ?", (book_id,))
        with patch("builtins.input", return_value=str(book_id)), \
             patch("builtins.print") as mock_print:
            borrow_book()
        output = " ".join(str(c) for c in mock_print.call_args_list)
        self.assertIn("out of stock", output)

    # -- delete_book ---------------------------------------------------------
    def test_delete_removes_book(self):
        self._insert_book(title="Delete Me")
        with patch("builtins.input", side_effect=["Delete Me", "y"]):
            delete_book()
        row = fetch_one("SELECT * FROM books WHERE title='Delete Me'")
        self.assertIsNone(row)

    def test_delete_cancel(self):
        self._insert_book(title="Keep Me")
        with patch("builtins.input", side_effect=["Keep Me", "n"]):
            delete_book()
        row = fetch_one("SELECT * FROM books WHERE title='Keep Me'")
        self.assertIsNotNone(row)

    # -- low_stock_report ----------------------------------------------------
    def test_low_stock_report(self):
        self._insert_book(title="Rare Book", quantity=1)
        with patch("builtins.print") as mock_print:
            low_stock_report()
        output = " ".join(str(c) for c in mock_print.call_args_list)
        self.assertIn("Rare Book", output)

    # -- category_report -----------------------------------------------------
    def test_category_report(self):
        self._insert_book(category="Science")
        with patch("builtins.print") as mock_print:
            category_report()
        output = " ".join(str(c) for c in mock_print.call_args_list)
        self.assertIn("Science", output)


# ══════════════════════════════════════════════
#  CLI / MAIN MENU  (from cli.py + muskan.py)
# ══════════════════════════════════════════════

def main():
    init_db()

    # ── Password gate ──────────────────────────
    print("\n Welcome to Library Book Tracker")
    while True:
        pwd = input("Enter password (or 'exit' to quit): ").strip()
        if pwd == APP_PASSWORD:
            break
        elif pwd.lower() == "exit":
            print("Goodbye!")
            return
        else:
            print("Wrong password. Try again.")

    # ── Main menu loop ─────────────────────────
    MENU = """
══════════════════════════════════
  LIBRARY BOOK TRACKER
══════════════════════════════════
 1. View All Books
 2. Add a Book
 3. Modify a Book
 4. Delete a Book
 5. Search Books
 6. Borrow a Book
 7. Return a Book
 8. Low Stock Report
 9. Category Report
10. Transaction Log
11. Exit
══════════════════════════════════"""

    actions = {
        "1":  view_all_books,
        "2":  add_book,
        "3":  modify_book,
        "4":  delete_book,
        "5":  search_books,
        "6":  borrow_book,
        "7":  return_book,
        "8":  low_stock_report,
        "9":  category_report,
        "10": transaction_log,
    }

    while True:
        print(MENU)
        choice = input("Choose an option: ").strip().split(".")[0].split(" ")[0]

        if choice == "11":
            print("Goodbye!")
            break
        elif choice in actions:
            actions[choice]()
        else:
            print("Invalid option. Please try again.")


# ══════════════════════════════════════════════
#  Entry point
# ══════════════════════════════════════════════

if __name__ == "__main__":
    if "--test" in sys.argv:
        # Run the built-in test suite
        sys.argv = [sys.argv[0]]   # strip --test so unittest doesn't choke
        unittest.main(verbosity=2)
    else:
        main()