using System;
using System.Collections.Generic;
using System.Globalization;
using System.IO;
using System.Net;
using System.Text;
using System.Threading;


public interface IShape
{
    string Name { get; }
    double Area();
    double Perimeter();
    string ParamsToString();
}


public struct Circle : IShape
{
    public double Radius { get; set; }
    public string Name => "Круг";

    public Circle(double radius) { Radius = radius; }

    public double Area() => Math.PI * Radius * Radius;
    public double Perimeter() => 2 * Math.PI * Radius;
    public string ParamsToString() => $"R = {Radius:F2}";

    public override string ToString() => $"Круг (R = {Radius:F2})";
}


public struct Rectangle : IShape
{
    public double Width { get; set; }
    public double Height { get; set; }
    public string Name => "Прямоугольник";

    public Rectangle(double width, double height)
    {
        Width = width;
        Height = height;
    }

    public double Area() => Width * Height;
    public double Perimeter() => 2 * (Width + Height);
    public string ParamsToString() => $"W = {Width:F2}, H = {Height:F2}";

    public override string ToString() => $"Прямоугольник (W = {Width:F2}, H = {Height:F2})";
}


public struct Triangle : IShape
{
    public double A { get; set; }
    public double B { get; set; }
    public double C { get; set; }
    public string Name => "Треугольник";

    public Triangle(double a, double b, double c)
    {
        A = a;
        B = b;
        C = c;
    }

    public double Perimeter() => A + B + C;

    // Формула Герона
    public double Area()
    {
        double s = Perimeter() / 2;
        return Math.Sqrt(s * (s - A) * (s - B) * (s - C));
    }

    public bool IsValid() =>
        A > 0 && B > 0 && C > 0 &&
        A + B > C && A + C > B && B + C > A;

    public string ParamsToString() => $"A = {A:F2}, B = {B:F2}, C = {C:F2}";

    public override string ToString() => $"Треугольник (A = {A:F2}, B = {B:F2}, C = {C:F2})";
}


public static class ShapeComparer
{
    public static int CompareByArea(IShape a, IShape b) => a.Area().CompareTo(b.Area());
    public static int CompareByPerimeter(IShape a, IShape b) => a.Perimeter().CompareTo(b.Perimeter());

    public static string GetComparisonResult(IShape a, IShape b)
    {
        var sb = new StringBuilder();
        int areaCmp = CompareByArea(a, b);
        int perimCmp = CompareByPerimeter(a, b);

        sb.AppendLine($"<h3>Сравнение: {a.Name} и {b.Name}</h3>");
        sb.AppendLine("<table>");
        sb.AppendLine("<tr><th>Параметр</th><th>" + a.Name + "</th><th>" + b.Name + "</th><th>Результат</th></tr>");

        sb.AppendLine($"<tr><td>Площадь</td><td>{a.Area():F2}</td><td>{b.Area():F2}</td><td>" +
            (areaCmp > 0 ? $"{a.Name} больше" : areaCmp < 0 ? $"{b.Name} больше" : "равны") + "</td></tr>");

        sb.AppendLine($"<tr><td>Периметр</td><td>{a.Perimeter():F2}</td><td>{b.Perimeter():F2}</td><td>" +
            (perimCmp > 0 ? $"{a.Name} больше" : perimCmp < 0 ? $"{b.Name} больше" : "равны") + "</td></tr>");

        sb.AppendLine("</table>");
        return sb.ToString();
    }
}


class ShapesWebServer
{
    // Хранилище созданных фигур (в памяти сервера)
    private static List<IShape> shapes = new List<IShape>();
    private static readonly object lockObj = new object();

    static void Main(string[] args)
    {
        int port = 8080;
        if (args.Length > 0 && int.TryParse(args[0], out int p)) port = p;

        
        lock (lockObj)
        {
            shapes.Add(new Circle(5));
            shapes.Add(new Rectangle(4, 6));
            shapes.Add(new Triangle(3, 4, 5));
        }

        var listener = new HttpListener();
        listener.Prefixes.Add($"http://localhost:{port}/");
        listener.Start();

        Console.WriteLine($"Сервер запущен: http://localhost:{port}/");
        Console.WriteLine("Нажмите Ctrl+C для остановки.\n");

        while (true)
        {
            try
            {
                var ctx = listener.GetContext();
                ThreadPool.QueueUserWorkItem(_ => HandleRequest(ctx));
            }
            catch (Exception ex)
            {
                Console.WriteLine("Ошибка: " + ex.Message);
            }
        }
    }

    
    static void HandleRequest(HttpListenerContext ctx)
    {
        try
        {
            var req = ctx.Request;
            var res = ctx.Response;
            string path = req.Url.AbsolutePath;
            var query = ParseQuery(req.Url.Query);

            // Обработка действий
            string action = query.ContainsKey("action") ? query["action"] : "";
            string message = "";

            if (action == "create")
                message = HandleCreate(query);
            else if (action == "clear")
            {
                lock (lockObj) shapes.Clear();
                message = "<p class='ok'>Список фигур очищен.</p>";
            }
            else if (action == "compare")
                message = HandleCompare(query);

            string html = RenderPage(message);

            byte[] buffer = Encoding.UTF8.GetBytes(html);
            res.ContentType = "text/html; charset=utf-8";
            res.ContentLength64 = buffer.Length;
            res.OutputStream.Write(buffer, 0, buffer.Length);
            res.OutputStream.Close();
        }
        catch (Exception ex)
        {
            try
            {
                byte[] err = Encoding.UTF8.GetBytes("<h1>Ошибка: " + ex.Message + "</h1>");
                ctx.Response.StatusCode = 500;
                ctx.Response.OutputStream.Write(err, 0, err.Length);
                ctx.Response.OutputStream.Close();
            }
            catch { }
        }
    }

    
    static string HandleCreate(Dictionary<string, string> q)
    {
        try
        {
            string type = q.ContainsKey("type") ? q["type"] : "";

            switch (type)
            {
                case "circle":
                {
                    double r = ParseDouble(q, "radius");
                    if (r <= 0) return "<p class='err'>Ошибка: радиус должен быть больше 0.</p>";
                    var c = new Circle(r);
                    AddShape(c);
                    return $"<p class='ok'>Создана фигура: {c}, площадь = {c.Area():F2}, периметр = {c.Perimeter():F2}</p>";
                }
                case "rectangle":
                {
                    double w = ParseDouble(q, "width");
                    double h = ParseDouble(q, "height");
                    if (w <= 0 || h <= 0) return "<p class='err'>Ошибка: стороны должны быть больше 0.</p>";
                    var rect = new Rectangle(w, h);
                    AddShape(rect);
                    return $"<p class='ok'>Создана фигура: {rect}, площадь = {rect.Area():F2}, периметр = {rect.Perimeter():F2}</p>";
                }
                case "triangle":
                {
                    double a = ParseDouble(q, "a");
                    double b = ParseDouble(q, "b");
                    double c = ParseDouble(q, "c");
                    var tri = new Triangle(a, b, c);
                    if (!tri.IsValid())
                        return "<p class='err'>Ошибка: треугольник с такими сторонами не существует.</p>";
                    AddShape(tri);
                    return $"<p class='ok'>Создана фигура: {tri}, площадь = {tri.Area():F2}, периметр = {tri.Perimeter():F2}</p>";
                }
                default:
                    return "<p class='err'>Неизвестный тип фигуры.</p>";
            }
        }
        catch (Exception ex)
        {
            return "<p class='err'>Ошибка ввода: " + ex.Message + "</p>";
        }
    }

    
    static string HandleCompare(Dictionary<string, string> q)
    {
        try
        {
            int i1 = int.Parse(q["i1"]);
            int i2 = int.Parse(q["i2"]);

            lock (lockObj)
            {
                if (i1 < 1 || i1 > shapes.Count || i2 < 1 || i2 > shapes.Count)
                    return "<p class='err'>Неверные номера фигур.</p>";
                if (i1 == i2)
                    return "<p class='err'>Выберите две разные фигуры.</p>";

                return ShapeComparer.GetComparisonResult(shapes[i1 - 1], shapes[i2 - 1]);
            }
        }
        catch
        {
            return "<p class='err'>Ошибка при сравнении фигур.</p>";
        }
    }

    
    static void AddShape(IShape s)
    {
        lock (lockObj) shapes.Add(s);
    }

    
    static double ParseDouble(Dictionary<string, string> q, string key)
    {
        if (!q.ContainsKey(key)) throw new Exception($"Параметр '{key}' не задан");
        return double.Parse(q[key].Replace(',', '.'), CultureInfo.InvariantCulture);
    }

    static Dictionary<string, string> ParseQuery(string query)
    {
        var dict = new Dictionary<string, string>();
        if (string.IsNullOrEmpty(query)) return dict;
        query = query.TrimStart('?');
        foreach (var pair in query.Split('&'))
        {
            if (string.IsNullOrEmpty(pair)) continue;
            var kv = pair.Split('=');
            string k = Uri.UnescapeDataString(kv[0]);
            string v = kv.Length > 1 ? Uri.UnescapeDataString(kv[1].Replace('+', ' ')) : "";
            dict[k] = v;
        }
        return dict;
    }

    static string RenderPage(string message)
    {
        var sb = new StringBuilder();
        sb.AppendLine("<!DOCTYPE html>");
        sb.AppendLine("<html lang='ru'><head><meta charset='utf-8'>");
        sb.AppendLine("<title>Геометрические фигуры — веб-версия</title>");
        sb.AppendLine("<style>");
        sb.AppendLine("body { font-family: Arial, sans-serif; margin: 30px; background: #f4f6f8; color: #2c3e50; }");
        sb.AppendLine("h1, h2 { color: #2c3e50; }");
        sb.AppendLine(".card { background:#fff; padding:20px; margin:15px 0; border-radius:8px; box-shadow:0 2px 6px rgba(0,0,0,.1); }");
        sb.AppendLine("table { border-collapse: collapse; margin: 10px 0; background: #fff; }");
        sb.AppendLine("th, td { border: 1px solid #999; padding: 8px 14px; text-align: center; }");
        sb.AppendLine("th { background: #3498db; color: #fff; }");
        sb.AppendLine("input, select, button { padding: 6px 10px; margin: 4px 0; font-size: 14px; }");
        sb.AppendLine("button { background:#3498db; color:#fff; border:none; border-radius:4px; cursor:pointer; }");
        sb.AppendLine("button:hover { background:#2980b9; }");
        sb.AppendLine(".ok  { color: #27ae60; font-weight: bold; }");
        sb.AppendLine(".err { color: #e74c3c; font-weight: bold; }");
        sb.AppendLine(".inline { display:inline-block; margin-right:10px; }");
        sb.AppendLine("</style></head><body>");

        sb.AppendLine("<h1>Приложение «Геометрические фигуры»</h1>");
        sb.AppendLine("<p>Веб-версия на C# / Mono. Ввод параметров, расчёт площади и периметра, сравнение фигур.</p>");

        
        if (!string.IsNullOrEmpty(message))
            sb.AppendLine("<div class='card'>" + message + "</div>");

        
        sb.AppendLine("<div class='card'>");
        sb.AppendLine("<h2>Создать фигуру</h2>");
        sb.AppendLine("<form method='get'>");
        sb.AppendLine("<input type='hidden' name='action' value='create'>");
        sb.AppendLine("<label>Тип фигуры: ");
        sb.AppendLine("<select name='type'>");
        sb.AppendLine("<option value='circle'>Круг</option>");
        sb.AppendLine("<option value='rectangle'>Прямоугольник</option>");
        sb.AppendLine("<option value='triangle'>Треугольник</option>");
        sb.AppendLine("</select></label><br>");
        sb.AppendLine("Круг: <input type='text' name='radius' size='6' placeholder='R' class='inline'>");
        sb.AppendLine("Прямоугольник: W <input type='text' name='width' size='6' class='inline'> H <input type='text' name='height' size='6' class='inline'><br>");
        sb.AppendLine("Треугольник: A <input type='text' name='a' size='6' class='inline'> B <input type='text' name='b' size='6' class='inline'> C <input type='text' name='c' size='6' class='inline'><br>");
        sb.AppendLine("<button type='submit'>Рассчитать и добавить</button>");
        sb.AppendLine("</form>");
        sb.AppendLine("<p><i>Заполняйте только поля для выбранной фигуры. Дробная часть отделяется точкой.</i></p>");
        sb.AppendLine("</div>");

        
        sb.AppendLine("<div class='card'>");
        sb.AppendLine("<h2>Список фигур</h2>");
        lock (lockObj)
        {
            if (shapes.Count == 0)
            {
                sb.AppendLine("<p>Список пуст. Создайте фигуру выше.</p>");
            }
            else
            {
                sb.AppendLine("<table>");
                sb.AppendLine("<tr><th>№</th><th>Фигура</th><th>Параметры</th><th>Площадь</th><th>Периметр</th></tr>");
                for (int i = 0; i < shapes.Count; i++)
                {
                    var s = shapes[i];
                    sb.AppendLine($"<tr><td>{i + 1}</td><td>{s.Name}</td><td>{s.ParamsToString()}</td>" +
                                  $"<td>{s.Area():F2}</td><td>{s.Perimeter():F2}</td></tr>");
                }
                sb.AppendLine("</table>");
                sb.AppendLine("<form method='get'>");
                sb.AppendLine("<input type='hidden' name='action' value='clear'>");
                sb.AppendLine("<button type='submit'>Очистить список</button>");
                sb.AppendLine("</form>");
            }
        }
        sb.AppendLine("</div>");

        
        sb.AppendLine("<div class='card'>");
        sb.AppendLine("<h2>Сравнить две фигуры</h2>");
        lock (lockObj)
        {
            if (shapes.Count < 2)
            {
                sb.AppendLine("<p>Для сравнения нужно минимум две фигуры.</p>");
            }
            else
            {
                sb.AppendLine("<form method='get'>");
                sb.AppendLine("<input type='hidden' name='action' value='compare'>");
                sb.AppendLine("Фигура №1: <input type='number' name='i1' min='1' max='" + shapes.Count + "' value='1' class='inline'> ");
                sb.AppendLine("Фигура №2: <input type='number' name='i2' min='1' max='" + shapes.Count + "' value='2' class='inline'> ");
                sb.AppendLine("<button type='submit'>Сравнить</button>");
                sb.AppendLine("</form>");
            }
        }
        sb.AppendLine("</div>");

        sb.AppendLine("</body></html>");
        return sb.ToString();
    }
}
