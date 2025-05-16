# Ship-Maintenance-Repositories
// AuthContext.jsx
import { createContext, useContext, useState, useEffect } from 'react';

const AuthContext = createContext();

export const AuthProvider = ({ children }) => {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    // Check localStorage for existing session
    const storedUser = localStorage.getItem('currentUser');
    if (storedUser) {
      setUser(JSON.parse(storedUser));
    }
    setLoading(false);
  }, []);

  const login = (email, password) => {
    // Simulate authentication against hardcoded users
    const users = JSON.parse(localStorage.getItem('users')) || [];
    const foundUser = users.find(u => u.email === email && u.password === password);
    
    if (foundUser) {
      setUser(foundUser);
      localStorage.setItem('currentUser', JSON.stringify(foundUser));
      return true;
    }
    return false;
  };

  const logout = () => {
    setUser(null);
    localStorage.removeItem('currentUser');
  };

  return (
    <AuthContext.Provider value={{ user, login, logout, loading }}>
      {children}
    </AuthContext.Provider>
  );
};

export const useAuth = () => useContext(AuthContext);
// ShipsContext.jsx
import { createContext, useContext, useState, useEffect } from 'react';

const ShipsContext = createContext();

export const ShipsProvider = ({ children }) => {
  const [ships, setShips] = useState([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    const storedShips = localStorage.getItem('ships');
    if (storedShips) {
      setShips(JSON.parse(storedShips));
    }
    setLoading(false);
  }, []);

  const addShip = (ship) => {
    const newShip = { ...ship, id: `s${Date.now()}` };
    const updatedShips = [...ships, newShip];
    setShips(updatedShips);
    localStorage.setItem('ships', JSON.stringify(updatedShips));
  };

  const updateShip = (id, updatedData) => {
    const updatedShips = ships.map(ship => 
      ship.id === id ? { ...ship, ...updatedData } : ship
    );
    setShips(updatedShips);
    localStorage.setItem('ships', JSON.stringify(updatedShips));
  };

  const deleteShip = (id) => {
    const updatedShips = ships.filter(ship => ship.id !== id);
    setShips(updatedShips);
    localStorage.setItem('ships', JSON.stringify(updatedShips));
  };

  return (
    <ShipsContext.Provider value={{ ships, loading, addShip, updateShip, deleteShip }}>
      {children}
    </ShipsContext.Provider>
  );
};

export const useShips = () => useContext(ShipsContext);
// JobCalendar.jsx
import React, { useState } from 'react';
import { format, startOfMonth, endOfMonth, eachDayOfInterval, isSameDay } from 'date-fns';
import { useJobs } from '../contexts/JobsContext';

const JobCalendar = () => {
  const [currentMonth, setCurrentMonth] = useState(new Date());
  const { jobs } = useJobs();
  
  const monthStart = startOfMonth(currentMonth);
  const monthEnd = endOfMonth(currentMonth);
  const daysInMonth = eachDayOfInterval({ start: monthStart, end: monthEnd });
  
  const [selectedDate, setSelectedDate] = useState(null);
  
  const jobsOnSelectedDate = selectedDate 
    ? jobs.filter(job => isSameDay(new Date(job.scheduledDate), selectedDate))
    : [];

  return (
    <div className="calendar-container">
      <div className="calendar-header">
        <button onClick={() => setCurrentMonth(prev => new Date(prev.setMonth(prev.getMonth() - 1)))}>
          Previous
        </button>
        <h2>{format(currentMonth, 'MMMM yyyy')}</h2>
        <button onClick={() => setCurrentMonth(prev => new Date(prev.setMonth(prev.getMonth() + 1)))}>
          Next
        </button>
      </div>
      
      <div className="calendar-grid">
        {daysInMonth.map(day => (
          <div 
            key={day.toString()}
            className={`calendar-day ${isSameDay(day, selectedDate) ? 'selected' : ''}`}
            onClick={() => setSelectedDate(day)}
          >
            {format(day, 'd')}
            <div className="job-indicator">
              {jobs.filter(job => isSameDay(new Date(job.scheduledDate), day)).length > 0 && '⚙️'}
            </div>
          </div>
        ))}
      </div>
      
      {selectedDate && (
        <div className="day-jobs">
          <h3>Jobs on {format(selectedDate, 'MMMM d, yyyy')}</h3>
          {jobsOnSelectedDate.length > 0 ? (
            <ul>
              {jobsOnSelectedDate.map(job => (
                <li key={job.id}>{job.type} - {job.priority} priority</li>
              ))}
            </ul>
          ) : (
            <p>No jobs scheduled</p>
          )}
        </div>
      )}
    </div>
  );
};
// NotificationCenter.jsx
import React, { useState, useEffect, useContext } from 'react';
import { useJobs } from '../contexts/JobsContext';
import { useAuth } from '../contexts/AuthContext';

const NotificationCenter = () => {
  const [notifications, setNotifications] = useState([]);
  const [isOpen, setIsOpen] = useState(false);
  const { jobs } = useJobs();
  const { user } = useAuth();

  useEffect(() => {
    // Load notifications from localStorage
    const storedNotifications = localStorage.getItem(`notifications_${user.id}`) || '[]';
    setNotifications(JSON.parse(storedNotifications));
    
    // Simulate real-time updates (in a real app, this would be from a WebSocket)
    const handleJobUpdate = (e) => {
      if (e.detail && e.detail.jobId) {
        const newNotification = {
          id: Date.now(),
          type: 'Job Update',
          message: `Job ${e.detail.jobId} has been updated`,
          read: false,
          timestamp: new Date().toISOString()
        };
        setNotifications(prev => {
          const updated = [newNotification, ...prev];
          localStorage.setItem(`notifications_${user.id}`, JSON.stringify(updated));
          return updated;
        });
      }
    };
    
    window.addEventListener('jobUpdated', handleJobUpdate);
    return () => window.removeEventListener('jobUpdated', handleJobUpdate);
  }, [user]);

  const markAsRead = (id) => {
    setNotifications(prev => {
      const updated = prev.map(n => 
        n.id === id ? { ...n, read: true } : n
      );
      localStorage.setItem(`notifications_${user.id}`, JSON.stringify(updated));
      return updated;
    });
  };

  const dismissNotification = (id) => {
    setNotifications(prev => {
      const updated = prev.filter(n => n.id !== id);
      localStorage.setItem(`notifications_${user.id}`, JSON.stringify(updated));
      return updated;
    });
  };

  return (
    <div className="notification-center">
      <button onClick={() => setIsOpen(!isOpen)}>
        🔔 {notifications.filter(n => !n.read).length > 0 && (
          <span className="badge">{notifications.filter(n => !n.read).length}</span>
        )}
      </button>
      
      {isOpen && (
        <div className="notification-dropdown">
          <div className="notification-header">
            <h3>Notifications</h3>
            <button onClick={() => setIsOpen(false)}>×</button>
          </div>
          
          {notifications.length === 0 ? (
            <p>No notifications</p>
          ) : (
            <ul>
              {notifications.map(notification => (
                <li 
                  key={notification.id} 
                  className={notification.read ? 'read' : 'unread'}
                >
                  <p>{notification.message}</p>
                  <small>{new Date(notification.timestamp).toLocaleString()}</small>
                  <div className="notification-actions">
                    {!notification.read && (
                      <button onClick={() => markAsRead(notification.id)}>Mark as read</button>
                    )}
                    <button onClick={() => dismissNotification(notification.id)}>Dismiss</button>
                  </div>
                </li>
              ))}
            </ul>
          )}
        </div>
      )}
    </div>
  );
};
