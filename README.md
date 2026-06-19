# Octal_grammer_Compiler 
package octal;

import java.util.*;
/**
 * OctalCompiler
 * -------------
 * A 3-phase compiler (Lexical, Syntax, Semantic) for the octal-number based
 * CFG defined in Octal_Grammer.docx.
 *
 * Terminals in the source program are octal numbers (digits 0-7, multi-digit
 * allowed: 10,11,...,75) and the signs '+' and '-'. Each number maps to a
 * Java-like keyword (e.g. 27 -> If, 25 -> For, 75 -> Integer).
 *
 * Because rule 75 -> Integer represents an integer literal, the user may
 * (and for semantic checks should) attach the actual integer value in
 * parentheses, e.g.  25 75(10)   means   for <int 10>.
 *
 * Usage:
 *     javac OctalCompiler.java
 *     java  OctalCompiler
 *
 * Then type a program on one line (or multiple, end with an empty line).
 * Example accepted programs:
 *     27 75(5)              -> if (5)
 *     27 75(1) 16 75(2)     -> if (1) else (2)
 *     25 75(10)             -> for (10)
 *     62 75(3)              -> while (3)
 *     44 75(0)              -> return 0
 *     73 74 75(7)           -> Modifier DataType Integer    (e.g. public int 7)
 */
public class Octal {
    // ---------------------------------------------------------------------
    // 1. KEYWORD / TERMINAL TABLE  (octal-code -> keyword name)
    // ---------------------------------------------------------------------
    static final Map<String, String> KEYWORDS = new LinkedHashMap<>();
    static {
        KEYWORDS.put("1",  "Abstract");
        KEYWORDS.put("2",  "Assert");
        KEYWORDS.put("3",  "Boolean");
        KEYWORDS.put("4",  "Break");
        KEYWORDS.put("5",  "Import");
        KEYWORDS.put("6",  "Case");
        KEYWORDS.put("7",  "Package");
        KEYWORDS.put("10", "Double");
        KEYWORDS.put("11", "Class");
        KEYWORDS.put("12", "Const");
        KEYWORDS.put("13", "Continue");
        KEYWORDS.put("14", "Default");
        KEYWORDS.put("15", "Do");
        KEYWORDS.put("16", "Else");
        KEYWORDS.put("17", "Catch");
        KEYWORDS.put("20", "Enum");
        KEYWORDS.put("21", "Extends");
        KEYWORDS.put("22", "Final");
        KEYWORDS.put("23", "Finally");
        KEYWORDS.put("24", "Float");
        KEYWORDS.put("25", "For");
        KEYWORDS.put("26", "Goto");
        KEYWORDS.put("27", "If");
        KEYWORDS.put("30", "Implements");
        KEYWORDS.put("33", "Int");
        KEYWORDS.put("34", "Interface");
        KEYWORDS.put("35", "Long");
        KEYWORDS.put("36", "Native");
        KEYWORDS.put("37", "New");
        KEYWORDS.put("41", "Private");
        KEYWORDS.put("42", "Protected");
        KEYWORDS.put("43", "Public");
        KEYWORDS.put("44", "Return");
        KEYWORDS.put("45", "Short");
        KEYWORDS.put("46", "Static");
        KEYWORDS.put("47", "Strictfp");
        KEYWORDS.put("50", "Super");
        KEYWORDS.put("51", "Switch");
        KEYWORDS.put("52", "Synchronized");
        KEYWORDS.put("54", "Throw");
        KEYWORDS.put("55", "Throws");
        KEYWORDS.put("56", "Transient");
        KEYWORDS.put("57", "Try");
        KEYWORDS.put("60", "Null");
        KEYWORDS.put("61", "Volatile");
        KEYWORDS.put("62", "While");
        KEYWORDS.put("66", "Byte");
        KEYWORDS.put("67", "Char");
        KEYWORDS.put("75", "Integer");
        // signs
        KEYWORDS.put("+",  "Sign");
        KEYWORDS.put("-",  "Sign");
    }
    // Convenience sets used by the parser
    static final Set<String> DATATYPE = new HashSet<>(Arrays.asList(
            "3","66","67","10","24","33","35","45","60"));
    static final Set<String> MODIFIER = new HashSet<>(Arrays.asList(
            "41","43","42","36","61","56","12","47","22","46"));
    // ---------------------------------------------------------------------
    // 2. TOKEN
    // ---------------------------------------------------------------------
    static class Token {
        final String code;     // e.g. "75", "27", "+"
        final String keyword;  // e.g. "Integer", "If", "Sign"
        final Integer value;   // populated only when code == "75" and user wrote 75(n)
        final int pos;         // 1-based index in stream
        Token(String c, String k, Integer v, int p){code=c;keyword=k;value=v;pos=p;}
    }
    // ---------------------------------------------------------------------
    // 3. LEXICAL ANALYZER
    // ---------------------------------------------------------------------
    static List<Token> tokenize(String src) {
        List<Token> out = new ArrayList<>();
        int i = 0, n = src.length(), idx = 1;
        while (i < n) {
            char c = src.charAt(i);
            if (Character.isWhitespace(c)) { i++; continue; }
            if (c == '+' || c == '-') {
                out.add(new Token(String.valueOf(c), "Sign", null, idx++));
                i++;
                continue;
            }
            if (c >= '0' && c <= '7') {
                // greedy: read digits 0-7
                int j = i;
                while (j < n && src.charAt(j) >= '0' && src.charAt(j) <= '7') j++;
                String code = src.substring(i, j);
                i = j;
                // optional integer literal: 75(123)
                Integer literal = null;
                if (code.equals("75") && i < n && src.charAt(i) == '(') {
                    int k = src.indexOf(')', i);
                    if (k < 0)
                        throw new RuntimeException(
                            "Lexical error at pos " + idx + ": unclosed '(' for integer literal");
                    String inside = src.substring(i + 1, k).trim();
                    try { literal = Integer.parseInt(inside); }
                    catch (NumberFormatException e) {
                        throw new RuntimeException(
                            "Lexical error at pos " + idx +
                            ": '" + inside + "' is not a valid integer literal");
                    }
                    i = k + 1;
                }
                String kw = KEYWORDS.get(code);
                if (kw == null)
                    throw new RuntimeException(
                        "Lexical error at pos " + idx +
                        ": '" + code + "' is not a defined terminal in the grammar");
                out.add(new Token(code, kw, literal, idx++));
                continue;
            }
            throw new RuntimeException(
                "Lexical error at pos " + idx + ": unexpected character '" + c + "'");
        }
        return out;
    }
    static void printTokenTable(List<Token> tokens) {
        System.out.println();
        System.out.println("=============== LEXICAL ANALYSIS — TOKEN TABLE ===============");
        System.out.printf("%-6s %-8s %-15s %-10s%n", "#", "Code", "Keyword", "Literal");
        System.out.println("--------------------------------------------------------------");
        for (Token t : tokens) {
            System.out.printf("%-6d %-8s %-15s %-10s%n",
                    t.pos, t.code, t.keyword,
                    (t.value == null ? "-" : t.value.toString()));
        }
        System.out.println("--------------------------------------------------------------");
        System.out.println("Total tokens: " + tokens.size());
    }
    // ---------------------------------------------------------------------
    // 4. SYNTAX ANALYZER  (recursive-descent for rule 71 -> Stmnt;
    //                       full input is 70 -> 71 70 | 71 , i.e. one or more
    //                       statements).
    // ---------------------------------------------------------------------
    static class Parser {
        final List<Token> toks;
        int p = 0;
        // record per-statement parses so semantic phase can re-walk them
        final List<List<Token>> statements = new ArrayList<>();
        Parser(List<Token> t){ toks = t; }
        boolean eof() { return p >= toks.size(); }
        Token peek() { return eof() ? null : toks.get(p); }
        Token eat()  { return toks.get(p++); }
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
                    "Syntax error: expected Integer (75) " + ctx +
                    (eof() ? " but reached end of input"
                           : " but found '" + toks.get(p).code + "'"));
            p++;
        }
        void parseProgram() {
            if (eof())
                throw new RuntimeException("Syntax error: empty program");
            while (!eof()) {
                int start = p;
                parseStmnt();
                statements.add(new ArrayList<>(toks.subList(start, p)));
            }
        }
        // 71 -> ... (Statement)
        void parseStmnt() {
            if (eof())
                throw new RuntimeException("Syntax error: expected statement");
            Token t = peek();
            switch (t.code) {
                case "27": // If
                    eat(); expectInteger("after 'If'");
                    if (!eof() && peek().code.equals("16")) { // Else
                        eat(); expectInteger("after 'Else'");
                    }
                    return;
                case "62": // While
                    eat(); expectInteger("after 'While'"); return;
                case "25": // For
                    eat(); expectInteger("after 'For'"); return;
                case "51": // Switch
                    eat(); expectInteger("after 'Switch'");
                    if (!eof() && peek().code.equals("6")) { // Case
                        eat(); expectInteger("after 'Case'");
                        if (!eof() && peek().code.equals("14")) { // Default
                            eat(); expectInteger("after 'Default'");
                        }
                    }
                    return;
                case "15": // Do
                    eat(); parseStmnt(); expect("62", "after 'Do <stmt>'");
                    expectInteger("after 'While' in do-while"); return;
                case "57": // Try
                    eat(); parseStmnt();
                    if (eof() || (!peek().code.equals("17") && !peek().code.equals("23")))
                        throw new RuntimeException(
                            "Syntax error: 'Try' statement must be followed by Catch(17) or Finally(23)");
                    eat(); expectInteger("after Catch/Finally"); return;
                case "44": // Return
                case "2":  // Assert
                case "55": // Throws
                case "54": // Throw
                    eat(); expectInteger("after '" + t.keyword + "'"); return;
                case "75": // bare Integer expression statement
                    eat(); return;
                default:
                    // Try declaration (72)
                    if (isDeclarationStart(t.code)) { parseDeclaration(); return; }
                    throw new RuntimeException(
                        "Syntax error at token " + t.pos +
                        ": unexpected '" + t.code + "' (" + t.keyword + ")");
            }
        }
        boolean isDeclarationStart(String code) {
            return MODIFIER.contains(code) || DATATYPE.contains(code) ||
                   code.equals("11") || code.equals("34") || code.equals("20") ||
                   code.equals("5")  || code.equals("7")  || code.equals("37") ||
                   code.equals("1");
        }
        // 72 -> Declaration
        void parseDeclaration() {
            Token t = peek();
            // Modifier (optional, may chain) then DataType + Integer  OR class/interface/etc.
            if (MODIFIER.contains(t.code)) {
                eat();
                if (!eof() && peek().code.equals("11")) { // Class
                    eat(); expectInteger("after 'Class'"); return;
                }
                if (!eof() && DATATYPE.contains(peek().code)) {
                    eat(); expectInteger("after DataType in declaration"); return;
                }
                // Modifier alone followed by Integer (e.g. final 5)
                expectInteger("after Modifier"); return;
            }
            if (DATATYPE.contains(t.code)) {
                eat(); expectInteger("after DataType"); return;
            }
            switch (t.code) {
                case "11": // Class
                case "34": // Interface
                case "20": // Enum
                case "5":  // Import
                case "7":  // Package
                case "37": // New
                    eat(); expectInteger("after '" + t.keyword + "'"); return;
                case "1":  // Abstract (may stand alone: 72 -> 1)
                    eat();
                    if (!eof() && peek().code.equals("11")) {
                        eat(); expectInteger("after 'Class'");
                    }
                    return;
                default:
                    throw new RuntimeException(
                        "Syntax error at token " + t.pos +
                        ": '" + t.code + "' cannot begin a declaration");
            }
        }
    }
    // ---------------------------------------------------------------------
    // 5. SEMANTIC ANALYZER
    //    - Every Integer (75) used in a control-flow / loop / branch context
    //      must carry a literal value (the user wrote 75(n)).
    //    - For-loop and While-loop counts must be >= 1.
    //    - Switch case value must be >= 0.
    //    - Return value must fit in a signed 32-bit int (always true here,
    //      but we report it as a successful semantic check).
    // ---------------------------------------------------------------------
    static void semanticCheck(List<List<Token>> statements) {
        int idx = 0;
        for (List<Token> stmt : statements) {
            idx++;
            String head = stmt.get(0).code;
            switch (head) {
                case "25": // For
                case "62": // While
                {
                    Token n = stmt.get(1);
                    requireLiteral(n, "loop count");
                    if (n.value <= 0)
                        throw new RuntimeException(
                            "Semantic error in statement " + idx +
                            ": loop count must be >= 1 (got " + n.value + ")");
                    break;
                }
                case "27": // If [/Else]
                {
                    requireLiteral(stmt.get(1), "if-condition");
                    if (stmt.size() >= 4 && stmt.get(2).code.equals("16"))
                        requireLiteral(stmt.get(3), "else-branch value");
                    break;
                }
                case "51": // Switch
                {
                    requireLiteral(stmt.get(1), "switch selector");
                    for (int i = 2; i < stmt.size(); i++) {
                        Token t = stmt.get(i);
                        if (t.code.equals("75")) {
                            requireLiteral(t, "case/default value");
                            if (t.value < 0)
                                throw new RuntimeException(
                                  "Semantic error in statement " + idx +
                                  ": case value must be >= 0 (got " + t.value + ")");
                        }
                    }
                    break;
                }
                case "44": // Return
                    requireLiteral(stmt.get(1), "return value");
                    break;
                default:
                    // declarations & others: literals optional
                    break;
            }
        }
    }
    static void requireLiteral(Token t, String role) {
        if (!t.code.equals("75") || t.value == null)
            throw new RuntimeException(
                "Semantic error: " + role +
                " requires an Integer literal written as 75(n)");
    }
    // 5b. EXECUTION  (interpret accepted program)
    //     - For/While 75(n)  -> prints 1..n
    //     - Do <stmt> While 75(n) -> repeats stmt n times (or prints 1..n if no inner action)
    //     - If 75(c) [Else 75(e)] -> prints c if c!=0 else e
    //     - Switch 75(s) Case 75(v) [Default 75(d)] -> prints v if s==v else d
    //     - Return 75(n) -> prints "Returned n"
    //     - Bare 75(n)  -> prints n
    // ---------------------------------------------------------------------
    static void execute(List<List<Token>> statements) {
        System.out.println("\n=============== EXECUTION OUTPUT ===============");
        boolean any = false;
        for (List<Token> stmt : statements) {
            String head = stmt.get(0).code;
            switch (head) {
                case "25":   // For
                case "62": { // While
                    int n = stmt.get(1).value;
                    System.out.print(stmt.get(0).keyword + " loop -> ");
                    for (int i = 1; i <= n; i++) System.out.print(i + (i == n ? "" : " "));
                    System.out.println();
                    any = true;
                    break;
                }
                case "15": { // Do <stmt> While 75(n)
                    // find the trailing 75 literal (last token)
                    Token last = stmt.get(stmt.size() - 1);
                    int n = (last.value != null) ? last.value : 1;
                    System.out.print("Do-While loop -> ");
                    for (int i = 1; i <= n; i++) System.out.print(i + (i == n ? "" : " "));
                    System.out.println();
                    any = true;
                    break;
                }
                case "27": { // If [Else]
                    int c = stmt.get(1).value;
                    if (c != 0) System.out.println("If branch -> " + c);
                    else if (stmt.size() >= 4) System.out.println("Else branch -> " + stmt.get(3).value);
                    else System.out.println("If branch -> (skipped, condition 0)");
                    any = true;
                    break;
                }
                case "51": { // Switch
                    int sel = stmt.get(1).value;
                    Integer matched = null, def = null;
                    for (int i = 2; i < stmt.size(); i++) {
                        if (stmt.get(i).code.equals("6") && i + 1 < stmt.size())
                            matched = stmt.get(i + 1).value;
                        if (stmt.get(i).code.equals("14") && i + 1 < stmt.size())
                            def = stmt.get(i + 1).value;
                    }
                    if (matched != null && matched == sel) System.out.println("Switch -> case " + matched);
                    else if (def != null) System.out.println("Switch -> default " + def);
                    else System.out.println("Switch -> no match");
                    any = true;
                    break;
                }
                case "44": // Return
                    System.out.println("Returned " + stmt.get(1).value);
                    any = true;
                    break;
                case "2":  // Assert
                case "54": // Throw
                case "55": // Throws
                    System.out.println(stmt.get(0).keyword + " " + stmt.get(1).value);
                    any = true;
                    break;
                case "75": // bare integer
                    if (stmt.get(0).value != null) {
                        System.out.println("Value -> " + stmt.get(0).value);
                        any = true;
                    }
                    break;
                default:
                    // declarations: print signature
                    StringBuilder sig = new StringBuilder("Declared: ");
                    for (Token t : stmt) sig.append(t.keyword)
                        .append(t.value != null ? "(" + t.value + ")" : "").append(' ');
                    System.out.println(sig.toString().trim());
                    any = true;
            }
        }
        if (!any) System.out.println("(no executable output)");
        System.out.println("================================================\n");
    }

    // ---------------------------------------------------------------------
    // 6. DRIVER
    // ---------------------------------------------------------------------
    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        System.out.println("Octal-Grammar Compiler  (type 'exit' to quit)");
        System.out.println("Enter a program, then a blank line to compile.\n");
        while (true) {
            System.out.println(">>> Program:");
            StringBuilder sb = new StringBuilder();
            while (sc.hasNextLine()) {
                String line = sc.nextLine();
                if (line.trim().equalsIgnoreCase("exit")) return;
                if (line.isEmpty()) break;
                sb.append(line).append(' ');
            }
            String src = sb.toString().trim();
            if (src.isEmpty()) { System.out.println("(no input)\n"); continue; }
            try {
                List<Token> tokens = tokenize(src);
                printTokenTable(tokens);
                Parser parser = new Parser(tokens);
                parser.parseProgram();
                System.out.println("\n[Syntax]   OK — grammar matches.");
                semanticCheck(parser.statements);
                System.out.println("[Semantic] OK — program is logically valid.");
                System.out.println("\n>>> RESULT: ACCEPTED  (" + parser.statements.size()
                                   + " statement(s))\n");
            } catch (RuntimeException ex) {
                System.out.println("\n>>> RESULT: REJECTED");
                System.out.println("    " + ex.getMessage() + "\n");
            }
        }
    }
}
