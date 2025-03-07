from flask import Flask, render_template, request, send_file
import matplotlib.pyplot as plt
from math import pi, sin, cos, sqrt
from matplotlib.animation import FuncAnimation
import numpy as np
import io

app = Flask(__name__)

# Yer çekimi ivmesi seçenekleri
gravity_options = {
    "Dünya": 9.81,
    "Ay": 1.62,
    "Mars": 3.71,
    "Jüpiter": 24.79,
    "Güneş": 274
}

@app.route("/", methods=["GET", "POST"])
def index():
    if request.method == "POST":
        # Kullanıcı girdilerini al
        L1 = float(request.form.get("L1", 2))
        L2 = float(request.form.get("L2", 2))
        m1 = float(request.form.get("m1", 1))
        m2 = float(request.form.get("m2", 1))
        Q1 = int(request.form.get("Q1", 45))
        Q2 = int(request.form.get("Q2", 45))
        gravity_choice = request.form.get("gravity", "Dünya")
        g = gravity_options.get(gravity_choice, 9.81)

        # Çift sarkaç hesaplamaları
        t0 = 0.
        t_max = 20.
        dt = 0.01
        n = int((t_max - t0) / dt)
        w1_0 = 0.0
        w2_0 = 0.0
        theta1 = Q1 * pi / 180
        theta2 = Q2 * pi / 180
        w1 = w1_0
        w2 = w2_0

        def theta1_double_dot(theta1, theta2, w1, w2):
            return (
                -g * (2 * m1 + m2) * sin(theta1)
                - m2 * g * sin(theta1 - 2 * theta2)
                - 2 * sin(theta1 - theta2) * m2 * (w2 ** 2 * L2 + w1 ** 2 * L1 * cos(theta1 - theta2))
            ) / (
                L1 * (2 * m1 + m2 - m2 * cos(2 * theta1 - 2 * theta2))
            )

        def theta2_double_dot(theta1, theta2, w1, w2):
            return (
                2 * sin(theta1 - theta2) * (
                w1 ** 2 * L1 * (m1 + m2)
                + g * (m1 + m2) * cos(theta1)
                + w2 ** 2 * L2 * m2 * cos(theta1 - theta2)
            )
            ) / (
                L2 * (2 * m1 + m2 - m2 * cos(2 * theta1 - 2 * theta2))
            )

        t_list = [t0]
        w1_list = [w1_0]
        w2_list = [w2_0]
        theta1_list = [theta1]
        theta2_list = [theta2]

        for i in range(n):
            t = t_list[i] + dt
            t_list.append(t)

            w1_new = w1_list[i] + dt * theta1_double_dot(theta1_list[i], theta2_list[i], w1_list[i], w2_list[i])
            w1_list.append(w1_new)

            w2_new = w2_list[i] + dt * theta2_double_dot(theta1_list[i], theta2_list[i], w1_list[i], w2_list[i])
            w2_list.append(w2_new)

            theta1_new = theta1_list[i] + w1_new * dt
            theta1_list.append(theta1_new)

            theta2_new = theta2_list[i] + w2_new * dt
            theta2_list.append(theta2_new)

        # Grafikleri ve animasyonu oluştur
        x1 = [L1 * sin(theta) for theta in theta1_list]
        y1 = [-L1 * cos(theta) for theta in theta1_list]
        x2 = [x1[i] + L2 * sin(theta2_list[i]) for i in range(len(x1))]
        y2 = [y1[i] - L2 * cos(theta2_list[i]) for i in range(len(y1))]

        # Grafikleri kaydet
        plt.figure(figsize=(8, 6))
        plt.plot(x1, y1, label='Pendulum 1', color='Red')
        plt.plot(x2, y2, label='Pendulum 2')
        plt.xlabel('x (m)')
        plt.ylabel('y (m)')
        plt.title('Double Pendulum Path')
        plt.legend()
        plt.grid(False)
        plt.savefig("static/path.png")
        plt.close()

        # Animasyonu kaydet
        fig, ax = plt.subplots(figsize=(8, 6))
        ax.set_xlim(-L1 - L2 - 0.5, L1 + L2 + 0.5)
        ax.set_ylim(-L1 - L2 - 0.5, L1 + L2 + 0.5)
        line, = ax.plot([], [], 'o-', lw=2)
        trace1, = ax.plot([], [], 'r-', alpha=0.5)
        trace2, = ax.plot([], [], 'b-', alpha=0.5)

        def init():
            line.set_data([], [])
            trace1.set_data([], [])
            trace2.set_data([], [])
            return line, trace1, trace2

        def update(frame):
            this_x = [0, x1[frame], x2[frame]]
            this_y = [0, y1[frame], y2[frame]]
            line.set_data(this_x, this_y)

            start = max(0, frame - 500)
            trace1.set_data(x1[start:frame], y1[start:frame])
            trace2.set_data(x2[start:frame], y2[start:frame])

            return line, trace1, trace2

        ani = FuncAnimation(fig, update, frames=range(0, len(x1), 10), init_func=init, blit=True, interval=25)
        ani.save("static/animation.gif", writer='ffmpeg', fps=30)
        plt.close()

        return render_template("result.html", path_image="path.png", animation="animation.gif")

    return render_template("index.html", gravity_options=gravity_options)

@app.route("/download")
def download():
    # Verileri indirme işlemi
    return send_file("static/animation.gif", as_attachment=True)

if __name__ == "__main__":
    app.run(debug=False)
