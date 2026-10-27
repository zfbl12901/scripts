#!/usr/bin/env python3
"""Génère un pom.xml factice à partir d'une liste de chemins Maven.

Usage: python gen_pom.py deps.txt [pom.xml]

- Les artefacts ayant un .jar -> <dependency> classique
- Les artefacts *-bom (pom seul) -> <dependencyManagement> scope import
- Les autres pom seuls (parents...) -> <dependency> type pom
"""
import sys
from xml.sax.saxutils import escape

src = sys.argv[1] if len(sys.argv) > 1 else "deps.txt"
dst = sys.argv[2] if len(sys.argv) > 2 else "pom.xml"

arts = {}  # (g, a, v) -> set(extensions)
for line in open(src, encoding="utf-8"):
    line = line.strip()
    if not line:
        continue
    parts = line.split("/")
    if len(parts) < 4:
        continue
    g, a, v = ".".join(parts[:-3]), parts[-3], parts[-2]
    ext = parts[-1].rsplit(".", 1)[-1]
    arts.setdefault((g, a, v), set()).add(ext)

def dep(g, a, v, indent, typ=None, scope=None):
    pad = " " * indent
    out = [f"{pad}<dependency>",
           f"{pad}    <groupId>{escape(g)}</groupId>",
           f"{pad}    <artifactId>{escape(a)}</artifactId>",
           f"{pad}    <version>{escape(v)}</version>"]
    if typ:
        out.append(f"{pad}    <type>{typ}</type>")
    if scope:
        out.append(f"{pad}    <scope>{scope}</scope>")
    out.append(f"{pad}</dependency>")
    return "\n".join(out)

boms, deps = [], []
for (g, a, v), exts in sorted(arts.items()):
    if "jar" in exts:
        deps.append(dep(g, a, v, 8))
    elif a.endswith("-bom"):
        boms.append(dep(g, a, v, 12, "pom", "import"))
    else:
        deps.append(dep(g, a, v, 8, "pom"))

pom = f"""<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.example</groupId>
    <artifactId>fake-deps-test</artifactId>
    <version>1.0.0-SNAPSHOT</version>
    <packaging>pom</packaging>

    <dependencyManagement>
        <dependencies>
{chr(10).join(boms)}
        </dependencies>
    </dependencyManagement>

    <dependencies>
{chr(10).join(deps)}
    </dependencies>
</project>
"""
open(dst, "w", encoding="utf-8").write(pom)
print(f"{dst} généré : {len(boms)} BOM, {len(deps)} dépendances")
