//
// QueryJSON.swift
// Mini JSON Query Engine: select / filter / projection on in-memory JSON
// Swift 5+
//

import Foundation

enum JSON {
    case obj([String: JSON])
    case arr([JSON])
    case str(String)
    case num(Double)
    case bool(Bool)
    case null
}

extension JSON: CustomStringConvertible {
    var description: String {
        switch self {
            case .obj(let d):
                let s = d.map { "\"\($0)\": \($1)" }.joined(separator: ", ")
                return "{\(s)}"
            case .arr(let a): return "[\(a.map { $0.description }.joined(separator: ", "))]"
            case .str(let s): return "\"\(s)\""
            case .num(let n): return "\(n)"
            case .bool(let b): return "\(b)"
            case .null: return "null"
        }
    }
}

// tiny parser for fixed JSON subset (demo only)
func parseJSON(_ s: String) -> JSON? {
    // For brevity, use Foundation's JSONSerialization then convert
    guard let data = s.data(using: .utf8) else { return nil }
    let obj = try? JSONSerialization.jsonObject(with: data, options: [])
    return convertAny(obj)
}

func convertAny(_ a: Any?) -> JSON {
    if a == nil { return .null }
    if let d = a as? [String: Any] {
        var m: [String: JSON] = [:]
        for (k,v) in d { m[k] = convertAny(v) }
        return .obj(m)
    } else if let arr = a as? [Any] {
        return .arr(arr.map { convertAny($0) })
    } else if let s = a as? String { return .str(s) }
    else if let n = a as? NSNumber {
        if CFGetTypeID(n) == CFBooleanGetTypeID() { return .bool(n.boolValue) }
        else { return .num(n.doubleValue) }
    } else { return .null }
}

// Query language (very small):
// Example: SELECT name,age FROM data WHERE age > 30
enum Expr {
    case field(String)
    case literal(JSON)
    case op(String, Expr, Expr)
}

struct Query {
    let select: [String]
    let whereExpr: Expr?
    // FROM is a root JSON array variable "data"
}

func evalExpr(_ expr: Expr, on row: JSON) -> JSON {
    switch expr {
        case .field(let f):
            if case .obj(let d) = row { return d[f] ?? .null }
            return .null
        case .literal(let v): return v
        case .op(let op, let a, let b):
            let av = evalExpr(a, on: row)
            let bv = evalExpr(b, on: row)
            return evalOp(op, av, bv)
    }
}

func evalOp(_ op: String, _ a: JSON, _ b: JSON) -> JSON {
    switch op {
        case ">":
            if case .num(let x) = a, case .num(let y) = b { return .bool(x > y) }
        case "<":
            if case .num(let x) = a, case .num(let y) = b { return .bool(x < y) }
        case "==":
            return .bool(a.description == b.description)
        case "&&":
            if case .bool(let x) = a, case .bool(let y) = b { return .bool(x && y) }
        default: break
    }
    return .null
}

// Tiny parser for a simple SELECT...WHERE grammar (very limited)
func parseQuery(_ q: String) -> Query? {
    // split SELECT ... FROM data WHERE ...
    let up = q.trimmingCharacters(in: .whitespacesAndNewlines)
    guard up.uppercased().hasPrefix("SELECT ") else { return nil }
    let parts = up.components(separatedBy: " WHERE ")
    let beforeWhere = parts[0]
    let wherePart = parts.count > 1 ? parts[1] : nil
    // parse select list
    guard let fromRange = beforeWhere.range(of: " FROM ") else { return nil }
    let sel = String(beforeWhere[beforeWhere.index(beforeWhere.startIndex, offsetBy:7)..<fromRange.lowerBound])
    let fields = sel.split(separator: ",").map { $0.trimmingCharacters(in: .whitespacesAndNewlines) }
    var whereExpr: Expr? = nil
    if let wp = wherePart {
        // very naive: assume "field op literal"
        let tokens = wp.split(separator: " ").map { String($0) }
        if tokens.count == 3 {
            let field = tokens[0]
            let op = tokens[1]
            let lit = tokens[2]
            let litJSON: JSON
            if let num = Double(lit) { litJSON = .num(num) } else { litJSON = .str(lit.replacingOccurrences(of: "\"", with: "")) }
            whereExpr = .op(op, .field(field), .literal(litJSON))
        }
    }
    return Query(select: fields, whereExpr: whereExpr)
}

func runQuery(_ qtxt: String, on data: JSON) -> [ [String: JSON] ] {
    guard let q = parseQuery(qtxt) else { return [] }
    guard case .arr(let rows) = data else { return [] }
    var out: [[String: JSON]] = []
    for row in rows {
        if let we = q.whereExpr {
            let res = evalExpr(we, on: row)
            if case .bool(let b) = res, b == false { continue }
        }
        var rec: [String: JSON] = [:]
        for f in q.select {
            let val = evalExpr(.field(f), on: row)
            rec[f] = val
        }
        out.append(rec)
    }
    return out
}

// Demo with sample JSON array
func demoQueryJSON() {
    print("=== QueryJSON Demo ===")
    let sample = """
    [
      {"name":"Alice","age":30,"score":95},
      {"name":"Bob","age":22,"score":66},
      {"name":"Carol","age":45,"score":88}
    ]
    """
    guard let data = parseJSON(sample) else { print("parse failed"); return }
    let q = "SELECT name,score FROM data WHERE age > 25"
    let results = runQuery(q, on: data)
    print("Results:")
    for r in results {
        print(r.mapValues { $0.description })
    }
}

demoQueryJSON()
