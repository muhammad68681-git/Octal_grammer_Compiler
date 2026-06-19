package octal;

import java.util.*;

/**
 * OctalCompiler v2.0
 * ------------------
 * Full compiler for the octal-number-based CFG from Octal_Grammer.docx.
 *
 * NEW in v2.0  — Expression / Arithmetic support
 * ------------------------------------------------
 * Any Integer position (75) may now carry a full arithmetic expression
 * in parentheses. Supported:
 *
 *   Literals      75(42)
 *   Variables     75(x)              -- must be declared first
 *   Arithmetic    75(3+4*2)          -- full precedence (+,-,*,/)
 *   Comparison    75(x>0)            -- returns 1 (true) or 0 (false)
 *   Unary         75(-x)   75(!x)
 *
 * Variable Declaration
 * --------------------
 *   33 75(n=<expr>)   -- declare  int  n = <expr>
 *   33 75(n)          -- declare  int  n = 0
 *   (any DataType code: 3,10,24,33,35,45,66,67,60)
 *
 * Statement forms (grammar rules → execution)
 * -------------------------------------------
 *   25 75(N)                   for  → prints 1 .. N
 *   62 75(N)                   while → prints 1 .. N
 *   15 71 62 75(N)             do-while → executes body N times
 *   27 75(C)                   if(C!=0) → prints C
 *   27 75(C) 16 75(E)          if/else
 *   51 75(S) 6 75(V) 14 75(D) switch/case/default
 *   44 75(V)                   return V
 *   2  75(V)                   assert V
 *   54 75(V)                   throw V
 *   55 75(V)                   throws V
 *   73 74 75(n=<expr>)         Modifier DataType n=<expr>  (declare+print)
 *   74 75(n=<expr>)            DataType n=<expr>
 *   75(<expr>)                 bare expression → evaluate and print
 *
 * Arithmetic-only statements  (no grammar code needed, just type the expression)
 *   75(3+4)       → prints 7
 *   75(x=5)       → assigns 5 to x, prints 5
 *   75(x*2)       → prints x*2
 *
 * Usage:
 *     javac octal/Octal.java
 *     java  octal.Octal
 */
public class Octal {

    // =========================================================================
    // 1. KEYWORD TABLE
    // =========================================================================
    static final Map<String, String> KEYWORDS = new LinkedHashMap<>();
    static {
        KEYWORDS.put("1",  "Abstract");   KEYWORDS.put("2",  "Assert");
        KEYWORDS.put("3",  "Boolean");    KEYWORDS.put("4",  "Break");
        KEYWORDS.put("5",  "Import");     KEYWORDS.put("6",  "Case");
        KEYWORDS.put("7",  "Package");    KEYWORDS.put("10", "Double");
        KEYWORDS.put("11", "Class");      KEYWORDS.put("12", "Const");
        KEYWORDS.put("13", "Continue");   KEYWORDS.put("14", "Default");
        KEYWORDS.put("15", "Do");         KEYWORDS.put("16", "Else");
        KEYWORDS.put("17", "Catch");      KEYWORDS.put("20", "Enum");
        KEYWORDS.put("21", "Extends");    KEYWORDS.put("22", "Final");
        KEYWORDS.put("23", "Finally");    KEYWORDS.put("24", "Float");
        KEYWORDS.put("25", "For");        KEYWORDS.put("26", "Goto");
        KEYWORDS.put("27", "If");         KEYWORDS.put("30", "Implements");
        KEYWORDS.put("33", "Int");        KEYWORDS.put("34", "Interface");
        KEYWORDS.put("35", "Long");       KEYWORDS.put("36", "Native");
        KEYWORDS.put("37", "New");        KEYWORDS.put("41", "Private");
        KEYWORDS.put("42", "Protected");  KEYWORDS.put("43", "Public");
        KEYWORDS.put("44", "Return");     KEYWORDS.put("45", "Short");
        KEYWORDS.put("46", "Static");     KEYWORDS.put("47", "Strictfp");
        KEYWORDS.put("50", "Super");      KEYWORDS.put("51", "Switch");
        KEYWORDS.put("52", "Synchronized"); KEYWORDS.put("54", "Throw");
        KEYWORDS.put("55", "Throws");     KEYWORDS.put("56", "Transient");
        KEYWORDS.put("57", "Try");        KEYWORDS.put("60", "Null");
        KEYWORDS.put("61", "Volatile");   KEYWORDS.put("62", "While");
        KEYWORDS.put("66", "Byte");       KEYWORDS.put("67", "Char");
        KEYWORDS.put("75", "Integer");
        KEYWORDS.put("+",  "Sign");       KEYWORDS.put("-",  "Sign");
    }
    static final Set<String> DATATYPE = new HashSet<>(Arrays.asList(
            "3","66","67","10","24","33","35","45","60"));
    static final Set<String> MODIFIER = new HashSet<>(Arrays.asList(
            "41","43","42","36","61","56","12","47","22","46"));

    // =========================================================================
    // 2. EXPRESSION EVALUATOR (mini recursive-descent over arithmetic strings)
    //    Supports: integer literals, variable names, +,-,*,/,%,==,!=,<,>,<=,>=,!
    //    Variable assignment inside expression: x=<expr>
    // =========================================================================
    static Map<String, Double> variables = new LinkedHashMap<>();

    /** Public entry: evaluate an expression string, return numeric result. */
    static double evalExpr(String s) {
        ExprParser ep = new ExprParser(s.trim());
        double v = ep.parseAssign();
        if (ep.pos < ep.src.length())
            throw new RuntimeException(
                "Expression error: unexpected '" + ep.src.charAt(ep.pos) + "' in \"" + s + "\"");
        return v;
    }

    static class ExprParser {
        final String src;
        int pos = 0;
        ExprParser(String s) { src = s; }

        void skip() { while (pos < src.length() && src.charAt(pos) == ' ') pos++; }

        // assignment:  name = expr  |  or|and
        double parseAssign() {
            skip();
            // check for identifier followed by '='
            int save = pos;
            String id = readIdentifier();
            skip();
            if (id != null && pos < src.length() && src.charAt(pos) == '='
                    && (pos + 1 >= src.length() || src.charAt(pos + 1) != '=')) {
                pos++;
                double val = parseAssign();
                variables.put(id, val);
                return val;
            }
            // rollback
            pos = save;
            return parseOr();
        }

        double parseOr() {
            double l = parseAnd();
            while (match("||")) l = ((l != 0) || (parseAnd() != 0)) ? 1 : 0;
            return l;
        }
        double parseAnd() {
            double l = parseEq();
            while (match("&&")) l = ((l != 0) && (parseEq() != 0)) ? 1 : 0;
            return l;
        }
        double parseEq() {
            double l = parseRel();
            skip();
            while (pos + 1 < src.length()) {
                if (match("==")) { l = (l == parseRel()) ? 1 : 0; }
                else if (match("!=")) { l = (l != parseRel()) ? 1 : 0; }
                else break;
                skip();
            }
            return l;
        }
        double parseRel() {
            double l = parseAdd();
            skip();
            while (pos < src.length()) {
                if (match("<=")) { l = (l <= parseAdd()) ? 1 : 0; }
                else if (match(">=")) { l = (l >= parseAdd()) ? 1 : 0; }
                else if (match("<"))  { l = (l <  parseAdd()) ? 1 : 0; }
                else if (match(">"))  { l = (l >  parseAdd()) ? 1 : 0; }
                else break;
                skip();
            }
            return l;
        }
        double parseAdd() {
            double l = parseMul();
            skip();
            while (pos < src.length()) {
                if (src.charAt(pos) == '+') { pos++; l += parseMul(); }
                else if (src.charAt(pos) == '-') { pos++; l -= parseMul(); }
                else break;
                skip();
            }
            return l;
        }
        double parseMul() {
            double l = parseUnary();
            skip();
            while (pos < src.length()) {
                if (src.charAt(pos) == '*') { pos++; l *= parseUnary(); }
                else if (src.charAt(pos) == '/') {
                    pos++;
                    double r = parseUnary();
                    if (r == 0) throw new RuntimeException("Semantic error: division by zero");
                    l /= r;
                } else if (src.charAt(pos) == '%') {
                    pos++;
                    double r = parseUnary();
                    if (r == 0) throw new RuntimeException("Semantic error: modulo by zero");
                    l %= r;
                } else break;
                skip();
            }
            return l;
        }
        double parseUnary() {
            skip();
            if (pos < src.length() && src.charAt(pos) == '-') { pos++; return -parseUnary(); }
            if (pos < src.length() && src.charAt(pos) == '!') { pos++; return (parseUnary() == 0) ? 1 : 0; }
            return parsePrimary();
        }
        double parsePrimary() {
            skip();
            if (pos >= src.length())
                throw new RuntimeException("Expression error: unexpected end of expression");
            char c = src.charAt(pos);
            // parenthesised sub-expression
            if (c == '(') {
                pos++;
                double v = parseAssign();
                skip();
                if (pos >= src.length() || src.charAt(pos) != ')')
                    throw new RuntimeException("Expression error: missing ')'");
                pos++;
                return v;
            }
            // numeric literal (integer or decimal)
            if (Character.isDigit(c)) {
                int start = pos;
                while (pos < src.length() && (Character.isDigit(src.charAt(pos)) || src.charAt(pos) == '.')) pos++;
                return Double.parseDouble(src.substring(start, pos));
            }
            // variable / identifier
            String id = readIdentifier();
            if (id != null) {
                if (!variables.containsKey(id))
                    throw new RuntimeException(
                        "Semantic error: variable '" + id + "' is not declared");
                return variables.get(id);
            }
            throw new RuntimeException("Expression error: unexpected character '" + c + "'");
        }

        /** Read a Java-style identifier starting at pos; return null if none. */
        String readIdentifier() {
            skip();
            if (pos >= src.length()) return null;
            char c = src.charAt(pos);
            if (!Character.isLetter(c) && c != '_') return null;
            int start = pos;
            while (pos < src.length() && (Character.isLetterOrDigit(src.charAt(pos)) || src.charAt(pos) == '_')) pos++;
            return src.substring(start, pos);
        }

        boolean match(String s) {
            skip();
            if (src.startsWith(s, pos)) { pos += s.length(); return true; }
            return false;
        }
    }

    /** Format a double: show as int if it is whole. */
    static String fmt(double v) {
        if (v == Math.floor(v) && !Double.isInfinite(v)) return String.valueOf((long) v);
        return String.valueOf(v);
    }

    // =========================================================================
    // 3. TOKEN
    // =========================================================================
    static class Token {
        final String  code;     // e.g. "75", "27", "+"
        final String  keyword;
        final String  rawExpr;  // raw content inside 75(...), null if absent
        final int     pos;      // 1-based

        Token(String c, String k, String expr, int p) {
            code = c; keyword = k; rawExpr = expr; pos = p;
        }

        /** Evaluate the expression inside this Integer token. */
        double eval() {
            if (!code.equals("75") || rawExpr == null)
                throw new RuntimeException(
                    "Semantic error at token " + pos +
                    ": Integer literal 75(expr) required");
            return evalExpr(rawExpr);
        }

        /** Evaluate but return int. */
        int evalInt() { return (int) eval(); }
    }

    // =========================================================================
    // 4. LEXER
    // =========================================================================
    static List<Token> tokenize(String src) {
        List<Token> out = new ArrayList<>();
        int i = 0, n = src.length(), idx = 1;
        while (i < n) {
            char c = src.charAt(i);
            if (Character.isWhitespace(c)) { i++; continue; }
            if (c == '+' || c == '-') {
                out.add(new Token(String.valueOf(c), "Sign", null, idx++));
                i++; continue;
            }
            if (c >= '0' && c <= '7') {
                int j = i;
                while (j < n && src.charAt(j) >= '0' && src.charAt(j) <= '7') j++;
                String code = src.substring(i, j);
                i = j;
                // optional (expr) — balanced parentheses
                String expr = null;
                if (code.equals("75") && i < n && src.charAt(i) == '(') {
                    int depth = 0, k = i;
                    while (k < n) {
                        if (src.charAt(k) == '(') depth++;
                        else if (src.charAt(k) == ')') { depth--; if (depth == 0) break; }
                        k++;
                    }
                    if (depth != 0)
                        throw new RuntimeException(
                            "Lexical error at pos " + idx + ": unclosed '(' for Integer expression");
                    expr = src.substring(i + 1, k).trim();
                    i = k + 1;
                }
                String kw = KEYWORDS.get(code);
                if (kw == null)
                    throw new RuntimeException(
                        "Lexical error at pos " + idx +
                        ": '" + code + "' is not a defined terminal");
                out.add(new Token(code, kw, expr, idx++));
                continue;
            }
            throw new RuntimeException(
                "Lexical error at pos " + idx +
                ": unexpected character '" + c + "'");
        }
        return out;
    }

    static void printTokenTable(List<Token> tokens) {
        System.out.println();
        System.out.println("============== LEXICAL ANALYSIS — TOKEN TABLE ==============");
        System.out.printf("%-5s %-8s %-18s %-20s%n", "#", "Code", "Keyword", "Expression");
        System.out.println("-------------------------------------------------------------");
        for (Token t : tokens) {
            System.out.printf("%-5d %-8s %-18s %-20s%n",
                    t.pos, t.code, t.keyword,
                    (t.rawExpr == null ? "-" : "(" + t.rawExpr + ")"));
        }
        System.out.println("-------------------------------------------------------------");
        System.out.println("Total tokens: " + tokens.size());
    }

    // =========================================================================
    // 5. PARSER  (recursive-descent)
    // =========================================================================
    static class Parser {
        final List<Token> toks;
        int p = 0;
        final List<List<Token>> statements = new ArrayList<>();
        Parser(List<Token> t) { toks = t; }
        boolean eof()  { return p >= toks.size(); }
        Token peek()   { return eof() ? null : toks.get(p); }
        Token eat()    { return toks.get(p++); }

        void expect(String code, String ctx) {
            if (eof() || !toks.get(p).code.equals(code))
                throw new RuntimeException(
                    "Syntax error: expected '" + code + "' (" +
                    KEYWORDS.getOrDefault(code, "?") + ") " + ctx +
                    (eof() ? " but reached end of input"
                           : " but found '" + toks.get(p).code + "'"));
            p++;
        }
        void expectInteger(String ctx) {
            if (eof() || !toks.get(p).code.equals("75"))
                throw new RuntimeException(
                    "Syntax error: expected Integer(75) " + ctx +
                    (eof() ? " but reached end of input"
                           : " but found '" + toks.get(p).code + "'"));
            p++;
        }

        void parseProgram() {
            if (eof()) throw new RuntimeException("Syntax error: empty program");
            while (!eof()) {
                int start = p;
                parseStmnt();
                statements.add(new ArrayList<>(toks.subList(start, p)));
            }
        }

        // 71 -> Stmnt
        void parseStmnt() {
            if (eof()) throw new RuntimeException("Syntax error: expected statement");
            Token t = peek();
            switch (t.code) {
                case "27":  // If [Else]
                    eat(); expectInteger("after 'If'");
                    if (!eof() && peek().code.equals("16")) { eat(); expectInteger("after 'Else'"); }
                    return;
                case "62":  // While
                    eat(); expectInteger("after 'While'"); return;
                case "25":  // For
                    eat(); expectInteger("after 'For'"); return;
                case "51":  // Switch [Case [Default]]
                    eat(); expectInteger("after 'Switch'");
                    if (!eof() && peek().code.equals("6")) {
                        eat(); expectInteger("after 'Case'");
                        if (!eof() && peek().code.equals("14")) { eat(); expectInteger("after 'Default'"); }
                    }
                    return;
                case "15":  // Do <stmt> While 75
                    eat(); parseStmnt();
                    expect("62", "after 'Do <stmt>'");
                    expectInteger("after 'While' in do-while"); return;
                case "57":  // Try <stmt> (Catch|Finally) 75
                    eat(); parseStmnt();
                    if (eof() || (!peek().code.equals("17") && !peek().code.equals("23")))
                        throw new RuntimeException(
                            "Syntax error: 'Try' must be followed by Catch(17) or Finally(23)");
                    eat(); expectInteger("after Catch/Finally"); return;
                case "44":  // Return 75
                case "2":   // Assert 75
                case "55":  // Throws 75
                case "54":  // Throw 75
                    eat(); expectInteger("after '" + t.keyword + "'"); return;
                case "75":  // bare Integer expression
                    eat(); return;
                default:
                    if (isDeclarationStart(t.code)) { parseDeclaration(); return; }
                    throw new RuntimeException(
                        "Syntax error at token " + t.pos +
                        ": unexpected '" + t.code + "' (" + t.keyword + ")");
            }
        }

        boolean isDeclarationStart(String code) {
            return MODIFIER.contains(code) || DATATYPE.contains(code)
                || code.equals("11") || code.equals("34") || code.equals("20")
                || code.equals("5")  || code.equals("7")  || code.equals("37")
                || code.equals("1");
        }

        // 72 -> Declaration
        void parseDeclaration() {
            Token t = peek();
            if (MODIFIER.contains(t.code)) {
                eat();
                if (!eof() && peek().code.equals("11")) { eat(); expectInteger("after Class"); return; }
                if (!eof() && DATATYPE.contains(peek().code)) { eat(); expectInteger("after DataType"); return; }
                expectInteger("after Modifier"); return;
            }
            if (DATATYPE.contains(t.code)) { eat(); expectInteger("after DataType"); return; }
            switch (t.code) {
                case "11": case "34": case "20":
                case "5":  case "7":  case "37":
                    eat(); expectInteger("after '" + t.keyword + "'"); return;
                case "1":
                    eat();
                    if (!eof() && peek().code.equals("11")) { eat(); expectInteger("after Class"); }
                    return;
                default:
                    throw new RuntimeException(
                        "Syntax error at token " + t.pos + ": '" + t.code + "' cannot begin a declaration");
            }
        }
    }

    // =========================================================================
    // 6. SEMANTIC CHECK
    //    Validates that required Integer tokens carry expressions,
    //    and checks loop / switch constraints.
    // =========================================================================
    static void semanticCheck(List<List<Token>> statements) {
        for (int idx = 0; idx < statements.size(); idx++) {
            List<Token> stmt = statements.get(idx);
            String head = stmt.get(0).code;
            switch (head) {
                case "25": case "62": {  // For / While — count must be >= 1
                    Token n = stmt.get(1);
                    requireExpr(n, "loop count", idx + 1);
                    double v = n.eval();
                    if (v < 1)
                        throw new RuntimeException(
                            "Semantic error in statement " + (idx+1) +
                            ": loop count must be >= 1 (got " + fmt(v) + ")");
                    break;
                }
                case "27": {  // If
                    requireExpr(stmt.get(1), "if-condition", idx + 1);
                    if (stmt.size() >= 4 && stmt.get(2).code.equals("16"))
                        requireExpr(stmt.get(3), "else-branch value", idx + 1);
                    break;
                }
                case "51": {  // Switch
                    requireExpr(stmt.get(1), "switch selector", idx + 1);
                    for (int i = 2; i < stmt.size(); i++) {
                        Token t = stmt.get(i);
                        if (t.code.equals("75")) {
                            requireExpr(t, "case/default value", idx + 1);
                            double v = t.eval();
                            if (v < 0)
                                throw new RuntimeException(
                                    "Semantic error in statement " + (idx+1) +
                                    ": case value must be >= 0 (got " + fmt(v) + ")");
                        }
                    }
                    break;
                }
                case "44":  // Return
                    requireExpr(stmt.get(1), "return value", idx + 1); break;
                default: break;
            }
        }
    }

    static void requireExpr(Token t, String role, int stmtIdx) {
        if (!t.code.equals("75") || t.rawExpr == null)
            throw new RuntimeException(
                "Semantic error in statement " + stmtIdx +
                ": " + role + " requires Integer with expression, e.g. 75(n)");
    }

    // =========================================================================
    // 7. EXECUTOR  — interprets each accepted statement
    // =========================================================================
    static void execute(List<List<Token>> statements) {
        System.out.println("\n=============== EXECUTION OUTPUT ===============");
        boolean any = false;
        for (List<Token> stmt : statements) {
            any |= execStmt(stmt);
        }
        if (!any) System.out.println("(no printable output)");
        System.out.println("================================================\n");
    }

    /** Execute one statement; return true if something was printed. */
    static boolean execStmt(List<Token> stmt) {
        String head = stmt.get(0).code;
        switch (head) {

            // ---- FOR  25 75(N) → print 1..N ---------------------------------
            case "25": {
                int n = stmt.get(1).evalInt();
                System.out.print("For loop (1 to " + n + ")  → ");
                for (int i = 1; i <= n; i++) System.out.print(i + (i == n ? "" : ", "));
                System.out.println();
                return true;
            }

            // ---- WHILE  62 75(N) → print 1..N -------------------------------
            case "62": {
                int n = stmt.get(1).evalInt();
                System.out.print("While loop (1 to " + n + ") → ");
                for (int i = 1; i <= n; i++) System.out.print(i + (i == n ? "" : ", "));
                System.out.println();
                return true;
            }

            // ---- DO-WHILE  15 <inner-stmt> 62 75(N) -------------------------
            case "15": {
                // find the trailing 75 (last token is the while-count)
                Token countTok = stmt.get(stmt.size() - 1);
                int n = (countTok.rawExpr != null) ? countTok.evalInt() : 1;
                // inner statement = tokens [1 .. size-3]  (exclude 15, 62, 75)
                List<Token> inner = stmt.subList(1, stmt.size() - 2);
                System.out.println("Do-While (repeat " + n + " time(s)):");
                for (int i = 0; i < n; i++) {
                    System.out.print("  Iteration " + (i + 1) + ": ");
                    execStmt(inner);
                }
                return true;
            }

            // ---- TRY  57 <inner-stmt> (17|23) 75(N) ------------------------
            case "57": {
                Token last = stmt.get(stmt.size() - 1);
                System.out.println("Try block:");
                List<Token> inner = stmt.subList(1, stmt.size() - 2);
                System.out.print("  Body: ");
                execStmt(inner);
                String handler = stmt.get(stmt.size() - 2).keyword;
                System.out.println("  " + handler + " handler value → "
                        + (last.rawExpr != null ? fmt(last.eval()) : "—"));
                return true;
            }

            // ---- IF  27 75(C) [16 75(E)] ------------------------------------
            case "27": {
                double cond = stmt.get(1).eval();
                if (cond != 0) {
                    System.out.println("If   condition=" + fmt(cond) + " → branch taken  → " + fmt(cond));
                } else if (stmt.size() >= 4 && stmt.get(2).code.equals("16")) {
                    double ev = stmt.get(3).eval();
                    System.out.println("If   condition=0 → else branch → " + fmt(ev));
                } else {
                    System.out.println("If   condition=0 → branch skipped");
                }
                return true;
            }

            // ---- SWITCH  51 75(S) [6 75(V)] [14 75(D)] ---------------------
            case "51": {
                double sel = stmt.get(1).eval();
                Double caseVal = null, defVal = null;
                for (int i = 2; i < stmt.size(); i++) {
                    if (stmt.get(i).code.equals("6") && i + 1 < stmt.size())
                        caseVal = stmt.get(i + 1).eval();
                    if (stmt.get(i).code.equals("14") && i + 1 < stmt.size())
                        defVal  = stmt.get(i + 1).eval();
                }
                System.out.print("Switch selector=" + fmt(sel) + " → ");
                if (caseVal != null && caseVal == sel) System.out.println("case " + fmt(caseVal) + " matched");
                else if (defVal != null)                System.out.println("default → " + fmt(defVal));
                else                                    System.out.println("no match");
                return true;
            }

            // ---- RETURN  44 75(V) -------------------------------------------
            case "44": {
                double v = stmt.get(1).eval();
                System.out.println("Return → " + fmt(v));
                return true;
            }

            // ---- ASSERT / THROW / THROWS  2|54|55  75(V) -------------------
            case "2": case "54": case "55": {
                double v = stmt.get(1).eval();
                System.out.println(stmt.get(0).keyword + " → " + fmt(v));
                return true;
            }

            // ---- BARE INTEGER  75(expr) -------------------------------------
            case "75": {
                Token t = stmt.get(0);
                if (t.rawExpr != null) {
                    double v = t.eval();
                    // if it was an assignment expression, mention the variable
                    String expr = t.rawExpr.trim();
                    int eq = expr.indexOf('=');
                    boolean isAssign = eq > 0 && (eq + 1 >= expr.length() || expr.charAt(eq + 1) != '=')
                                               && (eq == 0 || expr.charAt(eq - 1) != '!'
                                                           && expr.charAt(eq - 1) != '<'
                                                           && expr.charAt(eq - 1) != '>');
                    if (isAssign) {
                        String varName = expr.substring(0, eq).trim();
                        System.out.println("Assign: " + varName + " = " + fmt(v));
                    } else {
                        System.out.println("Value  → " + fmt(v));
                    }
                    return true;
                }
                return false;
            }

            // ---- DECLARATION  (DataType or Modifier DataType)  75(n=expr) ---
            default: {
                // Collect all tokens
                StringBuilder sig = new StringBuilder("Declare: ");
                for (Token t : stmt) {
                    if (t.code.equals("75")) {
                        if (t.rawExpr != null) {
                            double v = t.eval();
                            sig.append("(").append(t.rawExpr).append("=").append(fmt(v)).append(") ");
                        } else {
                            sig.append("Integer ");
                        }
                    } else {
                        sig.append(t.keyword).append(" ");
                    }
                }
                System.out.println(sig.toString().trim());
                return true;
            }
        }
    }

    // =========================================================================
    // 8. DRIVER
    // =========================================================================
    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        System.out.println("╔══════════════════════════════════════════════════════════╗");
        System.out.println("║         Octal-Grammar Compiler  v2.0                    ║");
        System.out.println("║  Type 'help' for syntax reference, 'exit' to quit.      ║");
        System.out.println("╚══════════════════════════════════════════════════════════╝");
        System.out.println();

        while (true) {
            // clear variable state between programs
            variables.clear();
            System.out.println(">>> Enter program (blank line to compile, 'exit' to quit):");
            StringBuilder sb = new StringBuilder();
            while (sc.hasNextLine()) {
                String line = sc.nextLine();
                if (line.trim().equalsIgnoreCase("exit")) return;
                if (line.trim().equalsIgnoreCase("help")) { printHelp(); break; }
                if (line.isEmpty()) break;
                sb.append(line).append(' ');
            }
            String src = sb.toString().trim();
            if (src.isEmpty()) { System.out.println("(no input)\n"); continue; }

            try {
                // PHASE 1 — Lexical
                List<Token> tokens = tokenize(src);
                printTokenTable(tokens);

                // PHASE 2 — Syntax
                Parser parser = new Parser(tokens);
                parser.parseProgram();
                System.out.println("\n[Syntax]   OK — " + parser.statements.size() + " statement(s) accepted.");

                // PHASE 3 — Semantic
                semanticCheck(parser.statements);
                System.out.println("[Semantic] OK — program is logically valid.");

                // PHASE 4 — Execute
                execute(parser.statements);

                System.out.println(">>> RESULT: ACCEPTED\n");
            } catch (RuntimeException ex) {
                System.out.println("\n>>> RESULT: REJECTED");
                System.out.println("    " + ex.getMessage() + "\n");
            }
        }
    }

    static void printHelp() {
        System.out.println();
        System.out.println("────────────────────────────────────────────────────────────");
        System.out.println("  OCTAL GRAMMAR QUICK REFERENCE");
        System.out.println("────────────────────────────────────────────────────────────");
        System.out.println("  Arithmetic inside 75(...):");
        System.out.println("    75(3+4*2)        → prints 11");
        System.out.println("    75(x=5)          → declares/assigns x=5, prints 5");
        System.out.println("    75(x*3)          → prints 15  (if x=5)");
        System.out.println("    75(10/2)         → prints 5");
        System.out.println("    75(7%3)          → prints 1");
        System.out.println();
        System.out.println("  Control flow:");
        System.out.println("    25 75(10)                    For  → 1,2,...,10");
        System.out.println("    62 75(5)                     While → 1,2,...,5");
        System.out.println("    15 75(1) 62 75(3)            Do inner While 3 times");
        System.out.println("    27 75(1) 16 75(99)           If/Else");
        System.out.println("    51 75(2) 6 75(2) 14 75(0)   Switch/Case/Default");
        System.out.println("    44 75(42)                    Return 42");
        System.out.println("    57 75(1) 17 75(0)            Try / Catch");
        System.out.println();
        System.out.println("  Declarations:");
        System.out.println("    33 75(n=10)                  int n = 10");
        System.out.println("    43 33 75(x=7)                public int x = 7");
        System.out.println("────────────────────────────────────────────────────────────");
        System.out.println();
    }
}
