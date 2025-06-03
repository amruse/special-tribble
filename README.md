import React, { useState, useEffect, useRef } from 'react';

const AdminPanel = () => {
  const [anime, setAnime] = useState({ title: '', category: '', tags: '', downloadLink: '', poster: null });
  const [animeList, setAnimeList] = useState([]);
  const [editingId, setEditingId] = useState(null);
  const [search, setSearch] = useState('');
  const [message, setMessage] = useState('');
  const [stats, setStats] = useState({ animeCount: 0, userCount: 0 });
  const [loadingStats, setLoadingStats] = useState(true);
  const [isUploading, setIsUploading] = useState(false);
  const posterRef = useRef(null);

  const fetchStats = async () => {
    try {
      const token = localStorage.getItem('token');
      const res = await fetch('/api/admin/stats', {
        headers: { Authorization: `Bearer ${token}` },
      });
      const data = await res.json();
      if (res.ok) setStats(data);
      else throw new Error();
    } catch {
      setMessage('❗ Error fetching stats');
    } finally {
      setLoadingStats(false);
    }
  };

  const fetchAnimeList = async () => {
    try {
      const res = await fetch('/api/anime');
      const data = await res.json();
      setAnimeList(data.reverse());
    } catch {
      setMessage('❗ Could not load anime list');
    }
  };

  useEffect(() => {
    fetchStats();
    fetchAnimeList();
  }, []);

  const handleChange = (e) => {
    const { name, value, files } = e.target;
    if (name === 'poster' && files?.[0]) {
      setAnime(prev => ({ ...prev, poster: files[0] }));
    } else {
      setAnime(prev => ({ ...prev, [name]: value }));
    }
  };

  const resetForm = () => {
    setAnime({ title: '', category: '', tags: '', downloadLink: '', poster: null });
    setEditingId(null);
    posterRef.current.value = null;
  };

  const handleSubmit = async (e) => {
    e.preventDefault();
    if (!anime.poster && !editingId) {
      setMessage('❗ Poster image required.');
      return;
    }

    setIsUploading(true);
    const formData = new FormData();
    Object.entries(anime).forEach(([k, v]) => v && formData.append(k, v));
    const token = localStorage.getItem('token');
    const url = editingId ? `/api/anime/${editingId}` : '/api/anime';
    const method = editingId ? 'PUT' : 'POST';

    try {
      const res = await fetch(url, {
        method,
        headers: { Authorization: `Bearer ${token}` },
        body: formData,
      });
      const data = await res.json();
      if (res.ok) {
        setMessage(editingId ? '✅ Anime updated!' : '✅ Anime uploaded!');
        fetchAnimeList();
        fetchStats();
        resetForm();
      } else {
        setMessage(data.message || '❗ Failed');
      }
    } catch {
      setMessage('❗ Submission error.');
    } finally {
      setIsUploading(false);
    }
  };

  const handleDelete = async (id) => {
    if (!window.confirm('Delete this anime?')) return;
    try {
      const token = localStorage.getItem('token');
      await fetch(`/api/anime/${id}`, {
        method: 'DELETE',
        headers: { Authorization: `Bearer ${token}` },
      });
      setMessage('✅ Deleted');
      fetchAnimeList();
      fetchStats();
    } catch {
      setMessage('❗ Delete failed');
    }
  };

  const handleEdit = (item) => {
    setEditingId(item._id);
    setAnime({
      title: item.title,
      category: item.category,
      tags: item.tags,
      downloadLink: item.downloadLink,
      poster: null, // re-upload if needed
    });
    window.scrollTo({ top: 0, behavior: 'smooth' });
  };

  return (
    <div className="min-h-screen bg-gradient-to-br from-black via-gray-900 to-black text-white font-['Poppins']">
      <div className="max-w-6xl mx-auto p-4 sm:p-6">

        {/* Upload Section */}
        <div className="bg-black bg-opacity-40 backdrop-blur-sm rounded-xl p-6 border border-gray-700 shadow-xl">
          <h1 className="text-3xl font-bold text-pink-500 text-center mb-4">RARETOONS ADMIN PANEL</h1>
          <div className="text-center mb-4">
            {loadingStats ? (
              <p className="text-cyan-400">Loading stats...</p>
            ) : (
              <p className="text-gray-300">
                🎥 <span className="text-pink-400">Animes:</span> {stats.animeCount} | 👥 <span className="text-cyan-400">Users:</span> {stats.userCount}
              </p>
            )}
          </div>
          <form onSubmit={handleSubmit} className="grid sm:grid-cols-2 gap-4">
            {['title', 'category', 'tags', 'downloadLink'].map((field) => (
              <input
                key={field}
                name={field}
                value={anime[field]}
                onChange={handleChange}
                required
                className="bg-gray-800 border border-gray-700 rounded p-2 text-white placeholder-gray-500 focus:ring-2 focus:ring-pink-500"
                placeholder={field === 'tags' ? 'Tags (comma separated)' : `Enter ${field}`}
              />
            ))}
            <input
              type="file"
              name="poster"
              ref={posterRef}
              onChange={handleChange}
              accept="image/*"
              className="sm:col-span-2 text-gray-300 file:py-2 file:px-4 file:rounded file:border-0 file:text-sm file:font-semibold file:bg-pink-500 file:text-white hover:file:bg-pink-600"
            />
            <button
              type="submit"
              disabled={isUploading}
              className="sm:col-span-2 w-full py-2 rounded-md text-white font-bold transition-all bg-gradient-to-r from-pink-500 to-cyan-500 hover:from-pink-600 hover:to-cyan-600"
            >
              {editingId ? 'Update Anime' : isUploading ? 'Uploading...' : 'Upload Anime'}
            </button>
          </form>
          {message && <p className="mt-4 text-center text-sm font-semibold text-pink-300">{message}</p>}
        </div>

        {/* Search & List */}
        <div className="mt-10">
          <input
            type="text"
            value={search}
            onChange={(e) => setSearch(e.target.value)}
            placeholder="🔍 Search anime title..."
            className="w-full bg-gray-800 border border-gray-700 rounded px-4 py-2 mb-4 text-white placeholder-gray-500"
          />

          <div className="grid sm:grid-cols-2 lg:grid-cols-3 gap-4">
            {animeList
              .filter((item) => item.title.toLowerCase().includes(search.toLowerCase()))
              .map((item) => (
                <div key={item._id} className="bg-black bg-opacity-30 p-4 rounded-xl border border-gray-700 shadow-lg">
                  <img src={item.posterUrl} alt={item.title} className="w-full h-48 object-cover rounded" />
                  <h2 className="text-lg font-semibold mt-2">{item.title}</h2>
                  <p className="text-sm text-gray-400">📂 {item.category} | 🏷 {item.tags}</p>
                  <div className="mt-3 flex justify-between">
                    <button onClick={() => handleEdit(item)} className="text-blue-400 hover:underline text-sm">Edit</button>
                    <button onClick={() => handleDelete(item._id)} className="text-red-400 hover:underline text-sm">Delete</button>
                  </div>
                </div>
              ))}
          </div>
        </div>

      </div>
    </div>
  );
};

export default AdminPanel;
# special-tribble
